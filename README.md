import streamlit as st
import sqlite3
import json
import os
import smtplib
from email.message import EmailMessage
from datetime import datetime, timedelta
from openai import OpenAI

# =====================================================
# CONFIGURATION
# =====================================================

st.set_page_config(
    page_title="CampusFlow AI",
    page_icon="🎓",
    layout="wide"
)

DB_NAME = "campusflow.db"

# =====================================================
# DATABASE
# =====================================================

def get_db():
    conn = sqlite3.connect(DB_NAME)
    conn.row_factory = sqlite3.Row
    return conn

def initialize_database():
    with get_db() as conn:
        conn.execute("""
            CREATE TABLE IF NOT EXISTS complaints (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                student_name TEXT NOT NULL,
                student_email TEXT,
                complaint TEXT NOT NULL,
                category TEXT,
                department TEXT,
                priority TEXT,
                status TEXT DEFAULT 'OPEN',
                created_at TEXT,
                followup_at TEXT,
                email_subject TEXT,
                email_body TEXT,
                email_sent INTEGER DEFAULT 0,
                reminder_sent INTEGER DEFAULT 0
            )
        """)

def create_ticket(args):
    """AI tool: create a complaint ticket."""

    allowed_departments = {
        "Scholarship": "Accounts",
        "Fees": "Accounts",
        "Hostel": "Hostel Office",
        "ID Card": "Administration",
        "Academics": "Academic Office",
        "Technical": "IT Support",
        "Other": "Administration"
    }

    category = args.get("category", "Other")

    if category not in allowed_departments:
        category = "Other"

    department = allowed_departments[category]

    priority = args.get("priority", "Medium")
    if priority not in ["Low", "Medium", "High"]:
        priority = "Medium"

    now = datetime.now()
    followup = now + timedelta(hours=48)

    with get_db() as conn:
        cursor = conn.execute("""
            INSERT INTO complaints (
                student_name,
                student_email,
                complaint,
                category,
                department,
                priority,
                status,
                created_at,
                followup_at
            )
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            args["student_name"],
            args.get("student_email", ""),
            args["complaint"],
            category,
            department,
            priority,
            "OPEN",
            now.isoformat(timespec="seconds"),
            followup.isoformat(timespec="seconds")
        ))

        ticket_id = cursor.lastrowid

    return {
        "success": True,
        "ticket_id": ticket_id,
        "department": department,
        "priority": priority,
        "status": "OPEN"
    }

def get_ticket(ticket_id):
    with get_db() as conn:
        return conn.execute(
            "SELECT * FROM complaints WHERE id = ?",
            (ticket_id,)
        ).fetchone()

def get_all_tickets():
    with get_db() as conn:
        return conn.execute(
            "SELECT * FROM complaints ORDER BY id DESC"
        ).fetchall()

def update_status(ticket_id, status):
    allowed = ["OPEN", "IN PROGRESS", "RESOLVED"]

    if status not in allowed:
        return

    with get_db() as conn:
        conn.execute("""
            UPDATE complaints
            SET status = ?
            WHERE id = ?
        """, (status, ticket_id))

def save_email(ticket_id, subject, body):
    with get_db() as conn:
        conn.execute("""
            UPDATE complaints
            SET email_subject = ?, email_body = ?
            WHERE id = ?
        """, (subject, body, ticket_id))

# =====================================================
# AI AGENT AND TOOL CALLING
# =====================================================

def run_agent(api_key, student_name, student_email, complaint):

    client = OpenAI(api_key=api_key)

    tools = [
        {
            "type": "function",
            "function": {
                "name": "create_ticket",
                "description": (
                    "Create a college complaint ticket after "
                    "understanding a new student complaint."
                ),
                "parameters": {
                    "type": "object",
                    "properties": {
                        "student_name": {
                            "type": "string"
                        },
                        "student_email": {
                            "type": "string"
                        },
                        "complaint": {
                            "type": "string"
                        },
                        "category": {
                            "type": "string",
                            "enum": [
                                "Scholarship",
                                "Fees",
                                "Hostel",
                                "ID Card",
                                "Academics",
                                "Technical",
                                "Other"
                            ]
                        },
                        "priority": {
                            "type": "string",
                            "enum": ["Low", "Medium", "High"]
                        }
                    },
                    "required": [
                        "student_name",
                        "student_email",
                        "complaint",
                        "category",
                        "priority"
                    ],
                    "additionalProperties": False
                }
            }
        }
    ]

    messages = [
        {
            "role": "system",
            "content": """
You are CampusFlow, an AI college complaint agent.

Your job:
1. Understand the student's complaint.
2. Identify its category and priority.
3. Create a ticket using the create_ticket tool.
4. Explain the result to the student.

Rules:
- For a new complaint, call create_ticket.
- Use only the student's supplied details.
- Never invent college policies or claim a real-world
  action happened unless the tool confirms it.
- Do not send emails yourself.
- Be concise and helpful.
"""
        },
        {
            "role": "user",
            "content": f"""
Student name: {student_name}
Student email: {student_email}

Complaint:
{complaint}

Analyze this complaint and create a ticket.
"""
        }
    ]

    ticket_id = None

    for _ in range(3):
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages,
            tools=tools,
            tool_choice="auto"
        )

        message = response.choices[0].message
        messages.append(message)

        if not message.tool_calls:
            return message.content or "Agent finished.", ticket_id

        for tool_call in message.tool_calls:
            if tool_call.function.name != "create_ticket":
                continue

            args = json.loads(tool_call.function.arguments)

            # Do not trust the model to alter student identity
            args["student_name"] = student_name
            args["student_email"] = student_email
            args["complaint"] = complaint

            result = create_ticket(args)
            ticket_id = result["ticket_id"]

            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result)
            })

    return "Ticket processing finished.", ticket_id

# =====================================================
# EMAIL SERVICE
# =====================================================

def send_email(
    sender,
    app_password,
    recipient,
    subject,
    body
):
    if not sender or not app_password or not recipient:
        raise ValueError(
            "Configure sender, App Password and recipient."
        )

    msg = EmailMessage()
    msg["From"] = sender
    msg["To"] = recipient
    msg["Subject"] = subject
    msg.set_content(body)

    with smtplib.SMTP_SSL("smtp.gmail.com", 465) as server:
        server.login(sender, app_password)
        server.send_message(msg)

def make_email(ticket):
    subject = (
        f"College Complaint #{ticket['id']} - "
        f"{ticket['category']}"
    )

    body = f"""
Dear {ticket['department']},

A student has submitted the following complaint.

Ticket ID: #{ticket['id']}
Student: {ticket['student_name']}
Category: {ticket['category']}
Priority: {ticket['priority']}

Complaint:
{ticket['complaint']}

Please review this issue and provide an update.

Regards,
CampusFlow AI
College Complaint Management System
""".strip()

    return subject, body

def mark_email_sent(ticket_id):
    with get_db() as conn:
        conn.execute("""
            UPDATE complaints
            SET email_sent = 1
            WHERE id = ?
        """, (ticket_id,))

# =====================================================
# REMINDERS
# =====================================================

def get_due_reminders():
    now = datetime.now().isoformat(timespec="seconds")

    with get_db() as conn:
        return conn.execute("""
            SELECT *
            FROM complaints
            WHERE followup_at <= ?
              AND status != 'RESOLVED'
              AND reminder_sent = 0
        """, (now,)).fetchall()

def mark_reminder_sent(ticket_id):
    with get_db() as conn:
        conn.execute("""
            UPDATE complaints
            SET reminder_sent = 1
            WHERE id = ?
        """, (ticket_id,))

# =====================================================
# USER INTERFACE
# =====================================================

initialize_database()

st.title("🎓 CampusFlow AI")
st.caption(
    "AI-powered college complaint management and automation"
)

with st.sidebar:
    st.header("⚙️ Configuration")

    api_key = st.text_input(
        "OpenAI API Key",
        type="password"
    )

    st.divider()
    st.subheader("📧 Gmail Settings")

    sender = st.text_input("Your Gmail address")

    app_password = st.text_input(
        "Gmail App Password",
        type="password"
    )

    st.caption(
        "Use a Gmail App Password, not your normal password."
    )

    st.divider()
    st.subheader("Department email addresses")

    accounts_email = st.text_input(
        "Accounts",
        placeholder="accounts@college.edu"
    )

    hostel_email = st.text_input(
        "Hostel Office",
        placeholder="hostel@college.edu"
    )

    admin_email = st.text_input(
        "Administration",
        placeholder="admin@college.edu"
    )

    academic_email = st.text_input(
        "Academic Office",
        placeholder="academics@college.edu"
    )

    it_email = st.text_input(
        "IT Support",
        placeholder="it@college.edu"
    )

department_emails = {
    "Accounts": accounts_email,
    "Hostel Office": hostel_email,
    "Administration": admin_email,
    "Academic Office": academic_email,
    "IT Support": it_email
}

tab1, tab2, tab3 = st.tabs([
    "📝 Submit Complaint",
    "📊 Dashboard",
    "⏰ Follow-ups"
])

# =====================================================
# TAB 1: NEW COMPLAINT
# =====================================================

with tab1:
    st.subheader("Submit a new complaint")

    with st.form("complaint_form"):
        student_name = st.text_input("Student name")
        student_email = st.text_input("Student email")

        complaint = st.text_area(
            "Describe your problem",
            placeholder=(
                "Example: My scholarship has been "
                "pending for 10 days."
            ),
            height=150
        )

        submitted = st.form_submit_button(
            "🤖 Analyze & Create Ticket",
            type="primary"
        )

    if submitted:
        if not student_name.strip() or not complaint.strip():
            st.error("Enter your name and complaint.")

        elif not api_key:
            st.error("Enter your OpenAI API key in the sidebar.")

        else:
            try:
                with st.spinner(
                    "AI agent is analyzing your complaint..."
                ):
                    result, ticket_id = run_agent(
                        api_key,
                        student_name.strip(),
                        student_email.strip(),
                        complaint.strip()
                    )

                st.success(result)

                if ticket_id:
                    ticket = get_ticket(ticket_id)

                    st.success(
                        f"🎫 Ticket #{ticket_id} created!"
                    )

                    col1, col2, col3 = st.columns(3)

                    col1.metric(
                        "Department",
                        ticket["department"]
                    )

                    col2.metric(
                        "Priority",
                        ticket["priority"]
                    )

                    col3.metric(
                        "Status",
                        ticket["status"]
                    )

                    st.info(
                        "An email draft can be reviewed "
                        "and sent from the Dashboard."
                    )

            except Exception as e:
                st.error(f"Error: {e}")

# =====================================================
# TAB 2: DASHBOARD
# =====================================================

with tab2:
    st.subheader("Complaint Dashboard")

    tickets = get_all_tickets()

    total = len(tickets)
    open_count = sum(
        t["status"] == "OPEN" for t in tickets
    )
    progress_count = sum(
        t["status"] == "IN PROGRESS" for t in tickets
    )
    resolved_count = sum(
        t["status"] == "RESOLVED" for t in tickets
    )

    c1, c2, c3, c4 = st.columns(4)

    c1.metric("Total", total)
    c2.metric("Open", open_count)
    c3.metric("In Progress", progress_count)
    c4.metric("Resolved", resolved_count)

    st.divider()

    if tickets:
        table_data = [
            {
                "Ticket": t["id"],
                "Student": t["student_name"],
                "Category": t["category"],
                "Department": t["department"],
                "Priority": t["priority"],
                "Status": t["status"],
                "Created": t["created_at"]
            }
            for t in tickets
        ]

        st.dataframe(
            table_data,
            use_container_width=True,
            hide_index=True
        )

        selected_id = st.selectbox(
            "Select ticket",
            options=[t["id"] for t in tickets],
            format_func=lambda x: f"Ticket #{x}"
        )

        ticket = get_ticket(selected_id)

        if ticket:
            st.markdown(f"### Ticket #{ticket['id']}")

            st.write("**Student:**", ticket["student_name"])
            st.write("**Complaint:**", ticket["complaint"])
            st.write("**Department:**", ticket["department"])
            st.write("**Priority:**", ticket["priority"])
            st.write("**Status:**", ticket["status"])

            new_status = st.selectbox(
                "Update status",
                ["OPEN", "IN PROGRESS", "RESOLVED"],
                index=[
                    "OPEN", "IN PROGRESS", "RESOLVED"
                ].index(ticket["status"])
            )

            if st.button("Update Ticket Status"):
                update_status(selected_id, new_status)
                st.success("Ticket status updated.")
                st.rerun()

            st.divider()
            st.subheader("📧 Email to Department")

            subject, body = make_email(ticket)

            subject = st.text_input(
                "Email subject",
                value=ticket["email_subject"] or subject,
                key=f"subject_{selected_id}"
            )

            body = st.text_area(
                "Email body",
                value=ticket["email_body"] or body,
                height=220,
                key=f"body_{selected_id}"
            )

            if st.button("Save Email Draft"):
                save_email(selected_id, subject, body)
                st.success("Email draft saved.")

            recipient = department_emails.get(
                ticket["department"], ""
            )

            st.write("**Recipient:**", recipient or "Not configured")

            if ticket["email_sent"]:
                st.success("Email marked as sent.")

            elif st.button(
                "✅ Approve & Send Email",
                type="primary"
            ):
                if not recipient:
                    st.error(
                        "Enter the department's email "
                        "address in the sidebar."
                    )

                elif not sender or not app_password:
                    st.error(
                        "Configure Gmail address and "
                        "App Password in the sidebar."
                    )

                else:
                    try:
                        send_email(
                            sender,
                            app_password,
                            recipient,
                            subject,
                            body
                        )

                        save_email(
                            selected_id,
                            subject,
                            body
                        )

                        mark_email_sent(selected_id)

                        st.success("Email sent successfully!")

                    except Exception as e:
                        st.error(f"Email error: {e}")

    else:
        st.info("No complaints have been submitted yet.")

# =====================================================
# TAB 3: FOLLOW-UPS
# =====================================================

with tab3:
    st.subheader("⏰ Automatic Follow-up Queue")

    st.write(
        "Tickets receive a follow-up deadline 48 hours "
        "after creation. Check this page to process "
        "reminders that are due."
    )

    if st.button("🔎 Check Due Reminders"):
        due = get_due_reminders()

        if not due:
            st.success("No reminders are due right now.")

        else:
            st.warning(f"{len(due)} reminder(s) are due.")

            for ticket in due:
                st.markdown(
                    f"### Ticket #{ticket['id']}"
                )

                st.write(
                    f"**Student:** {ticket['student_name']}"
                )

                st.write(
                    f"**Department:** {ticket['department']}"
                )

                st.write(
                    f"**Complaint:** {ticket['complaint']}"
                )

                st.write(
                    f"**Status:** {ticket['status']}"
                )

                recipient = department_emails.get(
                    ticket["department"], ""
                )

                reminder_subject = (
                    f"Reminder: Complaint #{ticket['id']}"
                )

                reminder_body = f"""
Dear {ticket['department']},

This is a follow-up regarding complaint #{ticket['id']}.

Student: {ticket['student_name']}
Complaint: {ticket['complaint']}
Current status: {ticket['status']}

Please provide an update.

Regards,
CampusFlow AI
""".strip()

                if not ticket["email_sent"]:
                    st.info(
                        "The original complaint email has "
                        "not been marked as sent."
                    )

                if st.button(
                    f"Send Reminder for #{ticket['id']}",
                    key=f"reminder_{ticket['id']}"
                ):
                    if not recipient or not sender or not app_password:
                        st.error(
                            "Configure Gmail and department "
                            "email settings first."
                        )
                    else:
                        try:
                            send_email(
                                sender,
                                app_password,
                                recipient,
                                reminder_subject,
                                reminder_body
                            )

                            mark_reminder_sent(ticket["id"])

                            st.success(
                                f"Reminder sent for "
                                f"ticket #{ticket['id']}."
                            )

                        except Exception as e:
                            st.error(f"Email error: {e}")

    st.divider()
    st.subheader("All unresolved complaints")

    unresolved = [
        t for t in get_all_tickets()
        if t["status"] != "RESOLVED"
    ]

    for ticket in unresolved:
        st.write(
            f"🎫 #{ticket['id']} — "
            f"{ticket['category']} — "
            f"{ticket['status']} — "
            f"Follow-up: {ticket['followup_at']}"
        )

st.divider()

st.caption(
    "CampusFlow AI | Hackathon Prototype | "
    "Review emails before sending."
)
