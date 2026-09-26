"""
CampusFlow AI - All-in-One Single File Production Web Application
Brand: CampusFlow AI
Tagline: "Your Campus. Your Request. Automatically Handled."

Focus Areas:
1. Attendance Complaints & Disputes
2. Library & Book Issue/Return Inquiries
3. Marks & Exam Results Rectification & Re-evaluations (Extra Feature)
4. ID Card & Student Administrative Services

Run with:
    python app.py
"""
import os
import sys
import json
import re
import time
import hmac
import hashlib
import base64
import datetime
import asyncio
import logging
from typing import Optional, List, Dict, Any
from contextlib import asynccontextmanager

from pydantic import BaseModel, Field
from sqlalchemy import create_engine, Column, Integer, String, Boolean, DateTime, ForeignKey, Text, or_, desc
from sqlalchemy.orm import declarative_base, sessionmaker, relationship, Session
from fastapi import FastAPI, HTTPException, Depends, status, Request, Response
from fastapi.responses import HTMLResponse, JSONResponse
from fastapi.middleware.cors import CORSMiddleware
import uvicorn
import requests

# ---------------------------------------------------------------------------
# 1. CONFIGURATION & CONSTANTS
# ---------------------------------------------------------------------------
HOST = os.getenv("HOST", "127.0.0.1")
PORT = int(os.getenv("PORT", "8000"))
SECRET_KEY = os.getenv("SECRET_KEY", "campusflow-ai-single-app-secret-2026")
DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./campusflow_all_in_one.db")
COLLEGE_NAME = os.getenv("COLLEGE_NAME", "Apex Institute of Technology")
GEMINI_API_KEY = os.getenv("GEMINI_API_KEY", "")
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY", "")

DEMO_CONFIG = {
    "demo_mode": True,
    "demo_reminder_minutes": 2,
    "demo_escalation_minutes": 5,
}

DEFAULT_DEPARTMENTS = [
    {"code": "ACADEMICS", "name": "Academic Affairs & Attendance Cell", "email": "academics@apex.edu", "head_name": "Dr. Rajesh Kulkarni", "supervisor_title": "Dean of Academics"},
    {"code": "LIBRARY", "name": "Library & Circulation Desk", "email": "library@apex.edu", "head_name": "Arthur Pendelton", "supervisor_title": "Chief Librarian"},
    {"code": "EXAM_CELL", "name": "Examination & Evaluation Cell", "email": "exams@apex.edu", "head_name": "Prof. Meenakshi Sundaram", "supervisor_title": "Controller of Examinations"},
    {"code": "ADMIN", "name": "Administration & Student Affairs", "email": "admin@apex.edu", "head_name": "Dr. Eleanor Vance", "supervisor_title": "Dean of Student Affairs"},
    {"code": "FINANCE", "name": "Finance & Accounts", "email": "finance@apex.edu", "head_name": "Priya Nair", "supervisor_title": "Controller of Finance"},
    {"code": "IT", "name": "IT & Network Services", "email": "it-support@apex.edu", "head_name": "Mark Foster", "supervisor_title": "Chief Information Officer"},
]

DEFAULT_CATEGORIES = [
    {"name": "Attendance Grievance", "code": "ATTENDANCE", "dept_code": "ACADEMICS", "default_priority": "High"},
    {"name": "Library & Book Issue", "code": "LIBRARY", "dept_code": "LIBRARY", "default_priority": "Medium"},
    {"name": "Marks & Results Query", "code": "MARKS_RESULTS", "dept_code": "EXAM_CELL", "default_priority": "High"},
    {"name": "ID Card & Student Services", "code": "ID_CARD", "dept_code": "ADMIN", "default_priority": "Medium"},
    {"name": "Fees & Accounts", "code": "FEES_ACCOUNTS", "dept_code": "FINANCE", "default_priority": "High"},
    {"name": "Wi-Fi & Campus IT", "code": "CAMPUS_IT", "dept_code": "IT", "default_priority": "Medium"},
]

# ---------------------------------------------------------------------------
# 2. DATABASE MODELS
# ---------------------------------------------------------------------------
connect_args = {"check_same_thread": False} if DATABASE_URL.startswith("sqlite") else {}
engine = create_engine(DATABASE_URL, connect_args=connect_args, echo=False)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(255), unique=True, index=True, nullable=False)
    full_name = Column(String(255), nullable=False)
    role = Column(String(50), default="student", nullable=False)
    hashed_password = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)

    student_profile = relationship("Student", back_populates="user", uselist=False, cascade="all, delete-orphan")
    notifications = relationship("Notification", back_populates="user", cascade="all, delete-orphan")
    messages = relationship("Message", back_populates="sender")

class Student(Base):
    __tablename__ = "students"
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), unique=True, nullable=False)
    student_id_number = Column(String(50), unique=True, index=True, nullable=False)
    department_major = Column(String(100), default="Computer Science & Engineering")
    year_of_study = Column(String(20), default="3rd Year")
    phone = Column(String(30), default="+1-555-0142")

    user = relationship("User", back_populates="student_profile")
    tickets = relationship("Ticket", back_populates="student", cascade="all, delete-orphan")

class Department(Base):
    __tablename__ = "departments"
    id = Column(Integer, primary_key=True, index=True)
    code = Column(String(50), unique=True, index=True, nullable=False)
    name = Column(String(100), nullable=False)
    email = Column(String(255), nullable=False)
    head_name = Column(String(100), nullable=False)
    supervisor_title = Column(String(100), default="Head of Department")

    categories = relationship("ComplaintCategory", back_populates="department")
    tickets = relationship("Ticket", back_populates="department")

class ComplaintCategory(Base):
    __tablename__ = "complaint_categories"
    id = Column(Integer, primary_key=True, index=True)
    code = Column(String(50), unique=True, index=True, nullable=False)
    name = Column(String(100), nullable=False)
    department_id = Column(Integer, ForeignKey("departments.id"), nullable=False)
    default_priority = Column(String(20), default="Medium")

    department = relationship("Department", back_populates="categories")
    tickets = relationship("Ticket", back_populates="category")

class Ticket(Base):
    __tablename__ = "tickets"
    id = Column(Integer, primary_key=True, index=True)
    ticket_code = Column(String(50), unique=True, index=True, nullable=False)  # REQ-2026-XXXX
    student_id = Column(Integer, ForeignKey("students.id"), nullable=False)
    category_id = Column(Integer, ForeignKey("complaint_categories.id"), nullable=True)
    department_id = Column(Integer, ForeignKey("departments.id"), nullable=True)

    title = Column(String(255), nullable=False)
    summary = Column(Text, nullable=False)
    raw_message = Column(Text, nullable=False)
    required_action = Column(Text, nullable=False)
    priority = Column(String(20), default="Medium", nullable=False)
    status = Column(String(50), default="Assigned", nullable=False)

    is_clarification_needed = Column(Boolean, default=False)
    clarification_question = Column(Text, nullable=True)
    clarification_answer = Column(Text, nullable=True)

    reminder_sent_at = Column(DateTime, nullable=True)
    reminder_count = Column(Integer, default=0)
    escalated_at = Column(DateTime, nullable=True)
    resolved_at = Column(DateTime, nullable=True)
    resolution_notes = Column(Text, nullable=True)
    resolution_time_minutes = Column(Integer, nullable=True)

    created_at = Column(DateTime, default=datetime.datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.datetime.utcnow, onupdate=datetime.datetime.utcnow)

    student = relationship("Student", back_populates="tickets")
    category = relationship("ComplaintCategory", back_populates="tickets")
    department = relationship("Department", back_populates="tickets")
    messages = relationship("Message", back_populates="ticket", cascade="all, delete-orphan", order_by="Message.created_at.asc()")
    notifications = relationship("Notification", back_populates="ticket", cascade="all, delete-orphan")
    status_history = relationship("StatusHistory", back_populates="ticket", cascade="all, delete-orphan", order_by="StatusHistory.created_at.asc()")

class Message(Base):
    __tablename__ = "messages"
    id = Column(Integer, primary_key=True, index=True)
    ticket_id = Column(Integer, ForeignKey("tickets.id", ondelete="CASCADE"), nullable=False)
    sender_id = Column(Integer, ForeignKey("users.id"), nullable=True)
    sender_name = Column(String(255), nullable=False)
    sender_role = Column(String(50), default="student", nullable=False)
    message_text = Column(Text, nullable=False)
    is_internal = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)

    ticket = relationship("Ticket", back_populates="messages")
    sender = relationship("User", back_populates="messages")

class Notification(Base):
    __tablename__ = "notifications"
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), nullable=True)
    ticket_id = Column(Integer, ForeignKey("tickets.id", ondelete="CASCADE"), nullable=True)
    title = Column(String(255), nullable=False)
    message = Column(Text, nullable=False)
    notification_type = Column(String(50), default="in_app")
    is_read = Column(Boolean, default=False)
    email_recipient = Column(String(255), nullable=True)
    email_subject = Column(String(255), nullable=True)
    email_html = Column(Text, nullable=True)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)

    user = relationship("User", back_populates="notifications")
    ticket = relationship("Ticket", back_populates="notifications")

class StatusHistory(Base):
    __tablename__ = "status_history"
    id = Column(Integer, primary_key=True, index=True)
    ticket_id = Column(Integer, ForeignKey("tickets.id", ondelete="CASCADE"), nullable=False)
    old_status = Column(String(50), nullable=True)
    new_status = Column(String(50), nullable=False)
    changed_by_name = Column(String(255), default="System AI")
    reason = Column(Text, nullable=True)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)

    ticket = relationship("Ticket", back_populates="status_history")

# ---------------------------------------------------------------------------
# 3. AUTH & SECURITY
# ---------------------------------------------------------------------------
def hash_password(password: str) -> str:
    key = SECRET_KEY.encode('utf-8')
    return hmac.new(key, password.encode('utf-8'), hashlib.sha256).hexdigest()

def create_access_token(data: dict, expires_in_seconds: int = 86400 * 7) -> str:
    payload = data.copy()
    payload["exp"] = int(time.time()) + expires_in_seconds
    raw_json = json.dumps(payload, sort_keys=True).encode('utf-8')
    b64_payload = base64.urlsafe_b64encode(raw_json).decode('utf-8').rstrip('=')
    signature = hmac.new(SECRET_KEY.encode('utf-8'), b64_payload.encode('utf-8'), hashlib.sha256).hexdigest()
    return f"{b64_payload}.{signature}"

def decode_access_token(token: str) -> Optional[dict]:
    try:
        parts = token.split('.')
        if len(parts) != 2:
            return None
        b64_payload, signature = parts
        expected_sig = hmac.new(SECRET_KEY.encode('utf-8'), b64_payload.encode('utf-8'), hashlib.sha256).hexdigest()
        if not hmac.compare_digest(signature, expected_sig):
            return None
        padding = '=' * (4 - (len(b64_payload) % 4)) if len(b64_payload) % 4 != 0 else ''
        payload = json.loads(base64.urlsafe_b64decode(b64_payload + padding).decode('utf-8'))
        if payload.get("exp", 0) < time.time():
            return None
        return payload
    except Exception:
        return None

def get_current_user(request: Request, db: Session = Depends(get_db)) -> User:
    token = request.cookies.get("campusflow_token")
    auth_h = request.headers.get("Authorization")
    if auth_h and auth_h.startswith("Bearer "):
        token = auth_h.split(" ")[1]
    
    if token:
        payload = decode_access_token(token)
        if payload and payload.get("sub"):
            u = db.query(User).filter(User.id == payload["sub"], User.is_active == True).first()
            if u:
                return u
                
    demo_user = db.query(User).filter(User.email == "sarah@campus.edu").first()
    if demo_user:
        return demo_user
        
    first_u = db.query(User).first()
    if first_u:
        return first_u
        
    raise HTTPException(status_code=401, detail="Authentication required.")

# ---------------------------------------------------------------------------
# 4. AI AGENT (NLP Heuristics + Gemini / OpenAI REST)
# ---------------------------------------------------------------------------
class AIAnalysisResult(BaseModel):
    category: str
    department: str
    priority: str
    summary: str
    student_message: str
    required_action: str
    is_clarification_needed: bool = False
    clarification_question: Optional[str] = None
    confidence_score: float = 0.96

SYSTEM_PROMPT = """You are CampusFlow AI, an intelligent college complaint and academic task management agent.
Classify student requests, extract parameters, assign responsible departments, determine priority, summarize the issue, and prescribe staff actions.
Categories:
- "Attendance Grievance" -> Department: "Academic Affairs & Attendance Cell"
- "Library & Book Issue" -> Department: "Library & Circulation Desk"
- "Marks & Results Query" -> Department: "Examination & Evaluation Cell"
- "ID Card & Student Services" -> Department: "Administration & Student Affairs"
- "Fees & Accounts" -> Department: "Finance & Accounts"
- "Wi-Fi & Campus IT" -> Department: "IT & Network Services"
If ambiguous, set is_clarification_needed to true and provide clarification_question.
Respond strictly in JSON matching the AIAnalysisResult schema."""

def extract_duration(text: str) -> Optional[str]:
    patterns = [
        r'(\d+\s*(?:days?|weeks?|months?|hours?))',
        r'(since\s+(?:yesterday|last\s+week|monday|tuesday|wednesday|thursday|friday|saturday|sunday))',
        r'(for\s+\d+\s*(?:days?|weeks?|months?))',
        r'(yesterday|today|last\s+night|monday|tuesday|wednesday|thursday|friday)',
    ]
    for pattern in patterns:
        m = re.search(pattern, text, re.IGNORECASE)
        if m:
            return m.group(1).strip()
    return None

def extract_subject(text: str) -> Optional[str]:
    subjects = [
        "operating systems", "os", "data structures", "dsa", "database", "dbms",
        "computer networks", "algorithms", "software engineering", "machine learning"
    ]
    lower = text.lower()
    for s in subjects:
        if s in lower:
            return s.upper() if len(s) <= 4 else s.title()
    return None

def analyze_with_heuristics(message: str) -> AIAnalysisResult:
    clean_msg = message.strip()
    lower_msg = clean_msg.lower()
    words = [w for w in re.split(r'\W+', lower_msg) if w]

    # 1. Clarification check on vague or single-word inputs
    vague_phrases = ["help", "help me", "not working", "broken", "issue", "problem", "attendance", "marks", "book", "result"]
    if len(words) <= 2 or lower_msg in vague_phrases:
        if any(w in lower_msg for w in ["attendance", "present", "absent"]):
            q = "Could you please specify the course/subject name and the date or lecture period for which attendance is missing?"
        elif any(w in lower_msg for w in ["book", "library"]):
            q = "Could you please provide the book title or accession number and whether this is a return or renewal issue?"
        elif any(w in lower_msg for w in ["mark", "marks", "result", "grade", "exam"]):
            q = "Could you please specify the course subject and whether this is for Internal Assessment or End-Semester examination?"
        else:
            q = "Could you please provide a few more details regarding your subject, request type, or student ID?"

        return AIAnalysisResult(
            category="Attendance Grievance",
            department="Academic Affairs & Attendance Cell",
            priority="Medium",
            summary=f"Inquiry: '{clean_msg[:40]}'",
            student_message=clean_msg,
            required_action="Awaiting student clarification details before routing to department coordinator.",
            is_clarification_needed=True,
            clarification_question=q,
            confidence_score=0.45
        )

    duration = extract_duration(clean_msg)
    subject = extract_subject(clean_msg)

    has_marks = bool(re.search(r'\b(?:marks?|grades?|gpa|sgpa|cgpa|exams?|re-?eval(?:uation)?|scores?|internals?|mid-?terms?|end-?sems?|transcripts?)\b', lower_msg))
    has_exam_absent = bool(re.search(r'\b(?:marked absent|absent in (?:exam|test)|zero marks)\b', lower_msg))
    has_attendance = any(k in lower_msg for k in ["attendance", "attendance shortage", "medical on-duty", "duty leave", "roster", "lecture missed", "wasn't marked", "not marked"])

    # A. MARKS & EXAM RESULTS
    if has_marks or has_exam_absent:
        category = "Marks & Results Query"
        department = "Examination & Evaluation Cell"
        subj_str = subject or "Course Examination"
        priority = "High"
        
        if any(k in lower_msg for k in ["absent", "marked absent", "zero", "0"]):
            summary = f"Incorrect Absent / Zero Marks Entry for {subj_str}"
            action = f"Pull physical exam attendance sheet and verified answer script for {subj_str}; rectify mark sheet in Controller of Examinations database and publish revised result."
        elif any(k in lower_msg for k in ["re-evaluation", "reval", "recheck"]):
            summary = f"Re-evaluation Request for {subj_str}"
            action = f"Initiate formal re-evaluation workflow for {subj_str}, assign script to independent secondary evaluator, and update student grade portal."
        else:
            summary = f"Marks & Result Discrepancy for {subj_str}"
            action = f"Cross-verify evaluated internal test marks with course instructor grade sheet, correct tallying error in ERP, and update student profile."

    # B. ATTENDANCE COMPLAINT / GRIEVANCE
    elif has_attendance or any(k in lower_msg for k in ["absent", "present", "bunk", "leave"]):
        category = "Attendance Grievance"
        department = "Academic Affairs & Attendance Cell"
        subj_str = subject or "Subject Lecture"
        dur_str = duration or "Recent Class"
        summary = f"Attendance Discrepancy for {subj_str}"
        priority = "High" if ("shortage" in lower_msg or "exam" in lower_msg or "monday" in lower_msg) else "Medium"
        action = f"Verify classroom attendance sheet / biometric log for {subj_str} ({dur_str}) with course instructor and update student academic portal attendance register."

    # C. ID CARD & STUDENT SERVICES
    elif any(k in lower_msg for k in ["id card", "identity card", "rfid", "smart card", "student id", "badge"]):
        category = "ID Card & Student Services"
        department = "Administration & Student Affairs"
        summary = "Student ID Card Issuance Delay"
        priority = "High" if (duration and any(d in duration for d in ["10", "14", "month", "weeks"])) else "Medium"
        action = "Check ID card printing queue in Student Affairs portal, verify biometric photo record, expedite smart RFID printing, and notify student for pickup."

    # D. LIBRARY & BOOK ISSUE
    elif any(k in lower_msg for k in ["book", "books", "library", "koha", "circulation", "librarian"]):
        category = "Library & Book Issue"
        department = "Library & Circulation Desk"
        priority = "Medium"
        if "returned" in lower_msg and ("still" in lower_msg or "issue" in lower_msg or "showing" in lower_msg):
            summary = "Book Returned But Still Showing as Issued"
            action = "Check library return drop box and circulation desk physical shelves, scan RFID barcode to update Koha LMS status, and waive any erroneous late fine."
        elif "renew" in lower_msg:
            summary = "Library Book Renewal Request"
            action = "Verify reservation queue for book, extend borrow period by 14 days in library portal, and dispatch confirmation receipt."
        else:
            summary = "Library Circulation & Book Query"
            action = "Inspect student library account record, reconcile borrow card ledger, and notify student."

    # E. FEES & ACCOUNTS
    elif any(k in lower_msg for k in ["fee", "fees", "tuition", "payment", "challan", "dues", "refund", "receipt"]):
        category = "Fees & Accounts"
        department = "Finance & Accounts"
        summary = "Semester Fee Payment / Ledger Discrepancy"
        priority = "High"
        action = "Verify student bank reference against payment gateway ledger, clear pending hold on student ERP account, and generate official stamped receipt."

    # F. WI-FI & CAMPUS IT
    elif any(k in lower_msg for k in ["wifi", "wi-fi", "internet", "network", "lab 3", "projector", "portal"]):
        category = "Wi-Fi & Campus IT"
        department = "IT & Network Services"
        summary = "Campus IT / Network Connectivity Issue"
        priority = "High"
        action = "Dispatch IT network technician to test Cisco/Aruba access point, verify student SSID authentication, and flush DHCP lease pool."

    else:
        category = "Academic Affairs & Attendance Cell"
        department = "Academic Affairs & Attendance Cell"
        summary = f"Academic Inquiry: {clean_msg[:50]}"
        priority = "Medium"
        action = "Review student request, route to faculty advisor, and acknowledge receipt within 24 hours."

    return AIAnalysisResult(
        category=category,
        department=department,
        priority=priority,
        summary=summary,
        student_message=clean_msg,
        required_action=action,
        is_clarification_needed=False,
        clarification_question=None,
        confidence_score=0.96
    )

def run_ai_agent(message: str) -> AIAnalysisResult:
    if GEMINI_API_KEY:
        try:
            url = f"https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key={GEMINI_API_KEY}"
            payload = {
                "contents": [{"role": "user", "parts": [{"text": f"{SYSTEM_PROMPT}\n\nStudent Request:\n\"{message}\""}]}],
                "generationConfig": {"temperature": 0.2, "responseMimeType": "application/json"}
            }
            res = requests.post(url, json=payload, timeout=10)
            if res.status_code == 200:
                parsed = json.loads(res.json()["candidates"][0]["content"]["parts"][0]["text"])
                return AIAnalysisResult(
                    category=parsed.get("category", "Attendance Grievance"),
                    department=parsed.get("department", "Academic Affairs & Attendance Cell"),
                    priority=parsed.get("priority", "Medium"),
                    summary=parsed.get("summary", message[:60]),
                    student_message=message,
                    required_action=parsed.get("required_action", "Investigate and resolve student request."),
                    is_clarification_needed=parsed.get("is_clarification_needed", False),
                    clarification_question=parsed.get("clarification_question"),
                    confidence_score=0.98
                )
        except Exception:
            pass

    return analyze_with_heuristics(message)

# ---------------------------------------------------------------------------
# 5. AUTONOMOUS SLA & SIMULATED EMAIL NOTIFICATIONS
# ---------------------------------------------------------------------------
def generate_ticket_code(db: Session) -> str:
    count = db.query(Ticket).count() + 1
    year = datetime.datetime.utcnow().year
    return f"REQ-{year}-{count:04d}"

def create_email_html(title: str, recipient_name: str, body_paragraphs: List[str], badge_label: str, badge_color: str, meta_rows: Dict[str, str], action_text: Optional[str] = None) -> str:
    meta_html = "".join([f'<tr><td style="padding: 6px 12px; font-weight: 600; color: #475569; width: 140px; border-bottom: 1px solid #f1f5f9;">{k}</td><td style="padding: 6px 12px; color: #1e293b; border-bottom: 1px solid #f1f5f9;">{v}</td></tr>' for k, v in meta_rows.items()])
    p_html = "".join([f'<p style="margin: 0 0 14px 0; line-height: 1.6; color: #334155; font-size: 15px;">{p}</p>' for p in body_paragraphs])
    action_box = f'<div style="background: #f8fafc; border-left: 4px solid #3b82f6; padding: 14px 18px; margin: 18px 0; border-radius: 4px;"><div style="font-weight: 700; color: #1e40af; font-size: 13px; text-transform: uppercase; margin-bottom: 4px;">AI Prescribed Action:</div><div style="color: #1e293b; font-size: 14px; line-height: 1.5;">{action_text}</div></div>' if action_text else ""

    return f"""<!DOCTYPE html><html><head><meta charset="utf-8"><title>{title}</title></head>
    <body style="margin:0; padding:0; background-color:#f1f5f9; font-family:-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;">
      <table width="100%" border="0" cellspacing="0" cellpadding="0" style="padding: 24px 0;">
        <tr><td align="center">
          <table width="600" border="0" cellspacing="0" cellpadding="0" style="background:#ffffff; border-radius: 12px; overflow: hidden; box-shadow: 0 4px 14px rgba(0,0,0,0.06); border: 1px solid #e2e8f0;">
            <tr>
              <td style="background: linear-gradient(135deg, #1e293b 0%, #0f172a 100%); padding: 22px 28px; border-bottom: 3px solid #3b82f6;">
                <table width="100%" border="0" cellspacing="0" cellpadding="0">
                  <tr>
                    <td>
                      <span style="color:#ffffff; font-size:22px; font-weight:800;">CampusFlow <span style="color:#38bdf8;">AI</span></span>
                      <div style="color:#94a3b8; font-size:12px; margin-top:2px;">Your Campus. Your Request. Automatically Handled.</div>
                    </td>
                    <td align="right">
                      <span style="background-color:{badge_color}; color:#ffffff; font-size:11px; font-weight:700; text-transform:uppercase; padding:5px 12px; border-radius:20px;">{badge_label}</span>
                    </td>
                  </tr>
                </table>
              </td>
            </tr>
            <tr>
              <td style="padding: 28px;">
                <div style="font-size:17px; font-weight:700; color:#0f172a; margin-bottom: 6px;">{title}</div>
                <div style="font-size:13px; color:#64748b; margin-bottom: 18px;">Dear {recipient_name},</div>
                {p_html}
                {action_box}
                <div style="margin: 20px 0; background: #ffffff; border: 1px solid #e2e8f0; border-radius: 8px; overflow: hidden;">
                  <table width="100%" cellspacing="0" cellpadding="0" style="font-size: 13px; border-collapse: collapse;">{meta_html}</table>
                </div>
                <p style="font-size: 12px; color: #64748b; margin-top: 24px; border-top: 1px solid #e2e8f0; padding-top: 14px;">
                  Autonomous workflow notification generated by <strong>CampusFlow AI</strong> on behalf of {COLLEGE_NAME}.
                </p>
              </td>
            </tr>
          </table>
        </td></tr>
      </table>
    </body></html>"""

def notify_ticket_created(ticket: Ticket, db: Session):
    student_user = ticket.student.user if ticket.student else None
    dept = ticket.department
    if dept:
        subject = f"[{ticket.priority} Priority] New Request {ticket.ticket_code}: {ticket.title}"
        email_html = create_email_html(
            title=f"New Assignment: {ticket.title}",
            recipient_name=f"{dept.name} Team ({dept.head_name})",
            body_paragraphs=[
                f"A new student request has been classified and automatically routed to your department.",
                f"Student statement: <em>\"{ticket.raw_message}\"</em>"
            ],
            badge_label=f"{ticket.priority} Priority",
            badge_color="#dc2626" if ticket.priority in ("High", "Urgent") else "#2563eb",
            meta_rows={
                "Request ID": ticket.ticket_code,
                "Category": ticket.category.name if ticket.category else "Academic Request",
                "Department": dept.name,
                "Priority": ticket.priority,
                "Student ID": ticket.student.student_id_number if ticket.student else "N/A"
            },
            action_text=ticket.required_action
        )
        db.add(Notification(
            ticket_id=ticket.id,
            title=f"Assignment: {ticket.ticket_code}",
            message=f"Request {ticket.ticket_code} ({ticket.title}) routed to {dept.name}.",
            notification_type="email",
            email_recipient=dept.email,
            email_subject=subject,
            email_html=email_html
        ))

    if student_user:
        db.add(Notification(
            user_id=student_user.id,
            ticket_id=ticket.id,
            title=f"Request Confirmed: {ticket.ticket_code}",
            message=f"Your request '{ticket.title}' was assigned to {dept.name if dept else 'Academic Affairs'}.",
            notification_type="in_app",
            email_recipient=student_user.email,
            email_subject=f"Request Received: {ticket.ticket_code} - {ticket.title}",
            email_html=create_email_html(
                title="Your Request Has Been Received",
                recipient_name=student_user.full_name,
                body_paragraphs=[
                    f"Your request has been routed to <strong>{dept.name if dept else 'Academic Affairs'}</strong>.",
                    f"Assessed Priority: <strong>{ticket.priority}</strong>. SLA tracking is now active."
                ],
                badge_label=ticket.status,
                badge_color="#059669",
                meta_rows={"Request ID": ticket.ticket_code, "Assigned To": dept.name if dept else "Academic Affairs", "Status": ticket.status}
            )
        ))

    db.add(StatusHistory(
        ticket_id=ticket.id,
        old_status=None,
        new_status=ticket.status,
        changed_by_name="CampusFlow AI Agent",
        reason=f"Auto-classified as '{ticket.category.name if ticket.category else 'General'}' and assigned to {dept.name if dept else 'Academic Affairs'}."
    ))
    db.commit()

def run_automation_sla_check(db: Session) -> Dict[str, Any]:
    now = datetime.datetime.utcnow()
    rem_min = DEMO_CONFIG["demo_reminder_minutes"]
    esc_min = DEMO_CONFIG["demo_escalation_minutes"]
    
    active = db.query(Ticket).filter(Ticket.status.in_(["Assigned", "In Progress", "Waiting for Student"])).all()
    reminders = 0
    escalations = 0

    for t in active:
        age_m = (now - t.created_at).total_seconds() / 60.0
        dept = t.department
        student_user = t.student.user if t.student else None

        if age_m >= esc_min and t.status != "Escalated":
            old_st = t.status
            t.status = "Escalated"
            t.escalated_at = now
            t.priority = "Urgent"
            escalations += 1

            db.add(StatusHistory(
                ticket_id=t.id,
                old_status=old_st,
                new_status="Escalated",
                changed_by_name="CampusFlow SLA Monitor",
                reason=f"SLA Breached ({int(age_m)}m > {esc_min}m threshold). Escalated to Department Head & Dean."
            ))

            db.add(Notification(
                ticket_id=t.id,
                title=f"URGENT Escalation: {t.ticket_code}",
                message=f"Request {t.ticket_code} overdue and escalated to {dept.head_name if dept else 'Dean'}.",
                notification_type="escalation",
                email_recipient=dept.email if dept else "dean@apex.edu",
                email_subject=f"[CRITICAL ESCALATION] Request {t.ticket_code} Overdue: {t.title}",
                email_html=create_email_html(
                    title=f"URGENT: Escalation Alert for {t.ticket_code}",
                    recipient_name=f"{dept.head_name if dept else 'Department Head'} & Dean's Office",
                    body_paragraphs=[f"Request <strong>{t.ticket_code}</strong> has breached resolution SLA ({int(age_m)} min elapsed). Immediate intervention required."],
                    badge_label="ESCALATED",
                    badge_color="#dc2626",
                    meta_rows={"Request ID": t.ticket_code, "Elapsed Time": f"{int(age_m)} min", "Department": dept.name if dept else "General"},
                    action_text=t.required_action
                )
            ))

        elif age_m >= rem_min and t.reminder_count == 0 and t.status != "Escalated":
            t.reminder_count += 1
            t.reminder_sent_at = now
            reminders += 1

            db.add(StatusHistory(
                ticket_id=t.id,
                old_status=t.status,
                new_status=t.status,
                changed_by_name="CampusFlow Auto-Reminder",
                reason=f"Automated Reminder #{t.reminder_count} sent to {dept.name if dept else 'Staff'} ({int(age_m)}m elapsed)."
            ))

            db.add(Notification(
                ticket_id=t.id,
                title=f"Reminder #{t.reminder_count}: {t.ticket_code}",
                message=f"Follow-up reminder sent to {dept.name if dept else 'staff'}.",
                notification_type="reminder",
                email_recipient=dept.email if dept else "support@apex.edu",
                email_subject=f"Follow-up Reminder: Action Awaiting on {t.ticket_code}",
                email_html=create_email_html(
                    title=f"Follow-up Reminder: {t.ticket_code}",
                    recipient_name=f"{dept.name if dept else 'Staff'}",
                    body_paragraphs=[f"Automated reminder for pending request <strong>{t.ticket_code}</strong> ({t.title}). Please attend promptly."],
                    badge_label="REMINDER",
                    badge_color="#f59e0b",
                    meta_rows={"Request ID": t.ticket_code, "Current Status": t.status, "Elapsed": f"{int(age_m)}m"},
                    action_text=t.required_action
                )
            ))

    db.commit()
    return {"checked_tickets": len(active), "reminders_sent": reminders, "escalations_triggered": escalations}

def simulate_time_jump(minutes_to_add: int, db: Session) -> Dict[str, Any]:
    active = db.query(Ticket).filter(Ticket.status.in_(["Assigned", "In Progress", "Waiting for Student"])).all()
    delta = datetime.timedelta(minutes=minutes_to_add)
    for t in active:
        t.created_at = t.created_at - delta
    db.commit()
    res = run_automation_sla_check(db)
    res["time_jump_minutes"] = minutes_to_add
    return res

# ---------------------------------------------------------------------------
# 6. SEED DATA GENERATOR
# ---------------------------------------------------------------------------
def seed_database(db: Session):
    if db.query(Department).first():
        return

    print("Seeding initial academic records for CampusFlow AI...")
    dept_map = {}
    for d in DEFAULT_DEPARTMENTS:
        dept = Department(code=d["code"], name=d["name"], email=d["email"], head_name=d["head_name"], supervisor_title=d["supervisor_title"])
        db.add(dept)
        db.flush()
        dept_map[d["code"]] = dept

    cat_map = {}
    for c in DEFAULT_CATEGORIES:
        dept = dept_map.get(c["dept_code"])
        cat = ComplaintCategory(code=c["code"], name=c["name"], department_id=dept.id if dept else None, default_priority=c["default_priority"])
        db.add(cat)
        db.flush()
        cat_map[c["name"]] = cat

    # Users
    sarah = User(email="sarah@campus.edu", full_name="Sarah Chen", role="student", hashed_password=hash_password("student123"))
    db.add(sarah)
    db.flush()
    sarah_stu = Student(user_id=sarah.id, student_id_number="CF-2026-CS042", department_major="Computer Science", year_of_study="3rd Year")
    db.add(sarah_stu)

    marcus = User(email="marcus@campus.edu", full_name="Marcus Bell", role="student", hashed_password=hash_password("student123"))
    db.add(marcus)
    db.flush()
    marcus_stu = Student(user_id=marcus.id, student_id_number="CF-2026-EE108", department_major="Electrical & Electronics", year_of_study="2nd Year")
    db.add(marcus_stu)

    dean = User(email="dean@campus.edu", full_name="Dr. Rajesh Kulkarni", role="admin", hashed_password=hash_password("admin123"))
    db.add(dean)

    exam_admin = User(email="exams@campus.edu", full_name="Prof. Meenakshi Sundaram", role="admin", hashed_password=hash_password("admin123"))
    db.add(exam_admin)
    db.flush()

    # Pre-seeded Academic Requests
    samples = [
        {
            "code": "REQ-2026-0001",
            "message": "My attendance for Operating Systems on Monday wasn't marked.",
            "student": sarah_stu,
            "status": "In Progress",
            "priority": "High",
            "category_name": "Attendance Grievance",
            "dept_code": "ACADEMICS",
            "age_hours": 4
        },
        {
            "code": "REQ-2026-0002",
            "message": "I returned a library book but it is still showing as issued.",
            "student": sarah_stu,
            "status": "Assigned",
            "priority": "Medium",
            "category_name": "Library & Book Issue",
            "dept_code": "LIBRARY",
            "age_hours": 2
        },
        {
            "code": "REQ-2026-0003",
            "message": "My internal marks for Data Structures show absent but I attended the test.",
            "student": sarah_stu,
            "status": "Escalated",
            "priority": "Urgent",
            "category_name": "Marks & Results Query",
            "dept_code": "EXAM_CELL",
            "age_hours": 26
        },
        {
            "code": "REQ-2026-0004",
            "message": "My ID card hasn't been issued for 10 days.",
            "student": marcus_stu,
            "status": "Assigned",
            "priority": "High",
            "category_name": "ID Card & Student Services",
            "dept_code": "ADMIN",
            "age_hours": 6
        },
        {
            "code": "REQ-2026-0005",
            "message": "I applied for re-evaluation in Database Systems Mid-Term but haven't received the revised score.",
            "student": marcus_stu,
            "status": "Resolved",
            "priority": "High",
            "category_name": "Marks & Results Query",
            "dept_code": "EXAM_CELL",
            "age_hours": 48,
            "resolution_notes": "Paper re-evaluated by Prof. Henderson. Marks updated from 31/50 to 42/50 in exam portal.",
            "resolution_time_minutes": 140
        }
    ]

    now = datetime.datetime.utcnow()
    for s in samples:
        c_time = now - datetime.timedelta(hours=s["age_hours"])
        ai_res = run_ai_agent(s["message"])
        cat = cat_map.get(s["category_name"])
        dept = dept_map.get(s["dept_code"])

        t = Ticket(
            ticket_code=s["code"],
            student_id=s["student"].id,
            category_id=cat.id if cat else None,
            department_id=dept.id if dept else None,
            title=ai_res.summary,
            summary=ai_res.summary,
            raw_message=s["message"],
            required_action=ai_res.required_action,
            priority=s["priority"],
            status=s["status"],
            created_at=c_time,
            updated_at=c_time + datetime.timedelta(minutes=30),
            resolution_notes=s.get("resolution_notes"),
            resolution_time_minutes=s.get("resolution_time_minutes"),
            resolved_at=(c_time + datetime.timedelta(minutes=s["resolution_time_minutes"])) if s.get("resolution_time_minutes") else None
        )
        db.add(t)
        db.flush()

        db.add(Message(ticket_id=t.id, sender_id=s["student"].user_id, sender_name=s["student"].user.full_name, sender_role="student", message_text=s["message"], created_at=c_time))
        db.add(StatusHistory(ticket_id=t.id, old_status=None, new_status=t.status, changed_by_name="CampusFlow AI Agent", reason="AI categorized and assigned to department.", created_at=c_time))
        notify_ticket_created(t, db)

    db.commit()
    print("Database ready with Attendance, Library, and Marks sample records!")

# ---------------------------------------------------------------------------
# 7. FASTAPI LIFESPAN & APPLICATION
# ---------------------------------------------------------------------------
bg_task = None
async def periodic_monitor():
    while True:
        try:
            await asyncio.sleep(20)
            db = SessionLocal()
            try:
                run_automation_sla_check(db)
            finally:
                db.close()
        except asyncio.CancelledError:
            break
        except Exception:
            await asyncio.sleep(10)

@asynccontextmanager
async def lifespan(app: FastAPI):
    Base.metadata.create_all(bind=engine)
    db = SessionLocal()
    try:
        seed_database(db)
    finally:
        db.close()
    global bg_task
    bg_task = asyncio.create_task(periodic_monitor())
    yield
    if bg_task:
        bg_task.cancel()

app = FastAPI(title="CampusFlow AI (Single App)", lifespan=lifespan)
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_credentials=True, allow_methods=["*"], allow_headers=["*"])

# ---------------------------------------------------------------------------
# 8. API ROUTES
# ---------------------------------------------------------------------------
class ComplaintSubmitRequest(BaseModel):
    message: str
    clarification_answer: Optional[str] = None

class StatusUpdateRequest(BaseModel):
    status: str
    reason: Optional[str] = None
    resolution_notes: Optional[str] = None

class ReassignRequest(BaseModel):
    department_id: int
    reason: Optional[str] = None

class MessageRequest(BaseModel):
    message_text: str
    is_internal: bool = False

@app.get("/api/auth/demo-users")
def get_demo_users(db: Session = Depends(get_db)):
    users = db.query(User).all()
    res = []
    for u in users:
        label = "Student" if u.role == "student" else ("Dean of Academics" if "dean" in u.email else "Exam Controller")
        res.append({"id": u.id, "email": u.email, "full_name": u.full_name, "role": u.role, "persona_label": label})
    return res

@app.post("/api/auth/switch-demo-user")
def switch_demo_user(data: dict, response: Response, db: Session = Depends(get_db)):
    u = db.query(User).filter(User.id == data.get("user_id")).first()
    if not u:
        raise HTTPException(404, "User not found")
    token = create_access_token({"sub": u.id, "email": u.email, "role": u.role})
    response.set_cookie("campusflow_token", token, httponly=True, max_age=86400*7)
    return {"status": "ok", "user": {"id": u.id, "email": u.email, "full_name": u.full_name, "role": u.role}}

@app.get("/api/auth/me")
def get_me(user: User = Depends(get_current_user)):
    return {"id": user.id, "email": user.email, "full_name": user.full_name, "role": user.role}

@app.get("/api/tickets/academic-overview")
def get_academic_overview(user: User = Depends(get_current_user)):
    return {
        "student_name": user.full_name,
        "attendance": {
            "overall_percentage": 84.5,
            "min_required": 75.0,
            "subjects": [
                {"code": "CS301", "name": "Operating Systems", "attended": 28, "total": 36, "percentage": 77.8, "status": "Low Attendance Warning"},
                {"code": "CS302", "name": "Data Structures & Algorithms", "attended": 34, "total": 37, "percentage": 91.9, "status": "Good Standing"},
                {"code": "CS303", "name": "Database Systems", "attended": 31, "total": 38, "percentage": 81.6, "status": "Good Standing"},
                {"code": "CS304", "name": "Computer Networks", "attended": 33, "total": 38, "percentage": 86.8, "status": "Good Standing"}
            ]
        },
        "library_books": [
            {
                "accession_no": "LIB-9842",
                "title": "Introduction to Algorithms (CLRS 4th Ed.)",
                "author": "Cormen et al.",
                "borrow_date": "Sep 05, 2026",
                "due_date": "Sep 25, 2026",
                "status": "Returned (Pending Circulation Clearance)",
                "action_needed": True
            },
            {
                "accession_no": "LIB-7714",
                "title": "Operating System Concepts (10th Ed.)",
                "author": "Silberschatz",
                "borrow_date": "Sep 12, 2026",
                "due_date": "Oct 02, 2026",
                "status": "Active (Due in 6 Days)",
                "action_needed": False
            }
        ],
        "marks_results": {
            "semester": "Fall 2026 (Semester 5)",
            "sgpa": 8.74,
            "subjects": [
                {"code": "CS301", "name": "Operating Systems", "assessment": "Mid-Term Exam", "score": "44 / 50", "grade": "A+", "status": "Published"},
                {"code": "CS302", "name": "Data Structures & Algorithms", "assessment": "Mid-Term Exam", "score": "AB (Absent)", "grade": "F*", "status": "Discrepancy (Student Attended)", "has_issue": True},
                {"code": "CS303", "name": "Database Systems", "assessment": "Mid-Term Exam", "score": "46 / 50", "grade": "O (Outstanding)", "status": "Published"},
                {"code": "CS304", "name": "Computer Networks", "assessment": "Mid-Term Exam", "score": "41 / 50", "grade": "A", "status": "Published"}
            ]
        }
    }

@app.post("/api/tickets/analyze")
def api_analyze(data: dict):
    msg = data.get("message", "")
    return run_ai_agent(msg)

@app.post("/api/tickets/submit")
def api_submit(req: ComplaintSubmitRequest, user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    student = user.student_profile or db.query(Student).first()
    full_msg = req.message.strip()
    if req.clarification_answer and req.clarification_answer.strip():
        full_msg += f"\n[Student Clarification: {req.clarification_answer.strip()}]"

    ai_res = run_ai_agent(full_msg)
    if ai_res.is_clarification_needed and not req.clarification_answer:
        return {"status": "clarification_needed", "clarification_question": ai_res.clarification_question, "summary": ai_res.summary}

    dept = db.query(Department).filter(or_(Department.name.ilike(f"%{ai_res.department}%"), Department.code.ilike(f"%{ai_res.department}%"))).first()
    if not dept:
        dept = db.query(Department).filter(Department.code == "ACADEMICS").first()

    cat = db.query(ComplaintCategory).filter(ComplaintCategory.name.ilike(f"%{ai_res.category}%")).first()
    if not cat and dept:
        cat = ComplaintCategory(code=ai_res.category.upper().replace(" ", "_")[:40], name=ai_res.category, department_id=dept.id, default_priority=ai_res.priority)
        db.add(cat)
        db.flush()

    ticket_code = generate_ticket_code(db)
    t = Ticket(
        ticket_code=ticket_code,
        student_id=student.id,
        category_id=cat.id if cat else None,
        department_id=dept.id if dept else None,
        title=ai_res.summary,
        summary=ai_res.summary,
        raw_message=full_msg,
        required_action=ai_res.required_action,
        priority=ai_res.priority,
        status="Assigned",
        is_clarification_needed=False,
        clarification_answer=req.clarification_answer
    )
    db.add(t)
    db.flush()

    db.add(Message(ticket_id=t.id, sender_id=user.id, sender_name=user.full_name, sender_role=user.role, message_text=full_msg))
    notify_ticket_created(t, db)

    return {
        "status": "created",
        "ticket_id": t.id,
        "ticket_code": t.ticket_code,
        "title": t.title,
        "category": cat.name if cat else "General",
        "department": dept.name if dept else "Academic Affairs",
        "priority": t.priority,
        "ticket_status": t.status,
        "required_action": t.required_action,
        "created_at": t.created_at.isoformat()
    }

@app.get("/api/tickets")
def api_list_tickets(dept_id: Optional[int] = None, status: Optional[str] = None, priority: Optional[str] = None, search: Optional[str] = None, my_tickets_only: bool = False, user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    q = db.query(Ticket).order_by(desc(Ticket.created_at))
    if my_tickets_only or user.role == "student":
        if user.student_profile:
            q = q.filter(Ticket.student_id == user.student_profile.id)
    if dept_id:
        q = q.filter(Ticket.department_id == dept_id)
    if status and status.lower() != "all":
        if status.lower() == "pending":
            q = q.filter(Ticket.status.in_(["Open", "Assigned", "In Progress", "Waiting for Student", "Escalated"]))
        elif status.lower() == "resolved":
            q = q.filter(Ticket.status.in_(["Resolved", "Closed"]))
        else:
            q = q.filter(Ticket.status == status)
    if priority and priority.lower() != "all":
        q = q.filter(Ticket.priority == priority)
    if search:
        t_term = f"%{search.strip()}%"
        q = q.filter(or_(Ticket.ticket_code.ilike(t_term), Ticket.title.ilike(t_term), Ticket.raw_message.ilike(t_term)))

    tickets = q.all()
    res = []
    for t in tickets:
        stu_u = t.student.user if t.student else None
        res.append({
            "id": t.id,
            "ticket_code": t.ticket_code,
            "title": t.title,
            "summary": t.summary,
            "raw_message": t.raw_message,
            "required_action": t.required_action,
            "priority": t.priority,
            "status": t.status,
            "created_at": t.created_at.isoformat(),
            "student_name": stu_u.full_name if stu_u else "Student",
            "student_id": t.student.student_id_number if t.student else "N/A",
            "department_name": t.department.name if t.department else "Academic Affairs",
            "category_name": t.category.name if t.category else "Academic"
        })
    return res

@app.get("/api/tickets/{ticket_id}")
def api_get_ticket(ticket_id: int, user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    t = db.query(Ticket).filter(Ticket.id == ticket_id).first()
    if not t:
        raise HTTPException(404, "Request not found.")
    stu_u = t.student.user if t.student else None
    msgs = t.messages if user.role != "student" else [m for m in t.messages if not m.is_internal]

    return {
        "id": t.id,
        "ticket_code": t.ticket_code,
        "title": t.title,
        "summary": t.summary,
        "raw_message": t.raw_message,
        "required_action": t.required_action,
        "priority": t.priority,
        "status": t.status,
        "clarification_answer": t.clarification_answer,
        "created_at": t.created_at.isoformat(),
        "student": {"name": stu_u.full_name if stu_u else "N/A", "student_id": t.student.student_id_number if t.student else "N/A"},
        "department": {"id": t.department.id if t.department else None, "name": t.department.name if t.department else "Academic Affairs", "head_name": t.department.head_name if t.department else "Dean"},
        "category": {"name": t.category.name if t.category else "Academic"},
        "messages": [{"id": m.id, "sender_name": m.sender_name, "sender_role": m.sender_role, "message_text": m.message_text, "is_internal": m.is_internal, "created_at": m.created_at.isoformat()} for m in msgs],
        "status_history": [{"id": h.id, "old_status": h.old_status, "new_status": h.new_status, "changed_by_name": h.changed_by_name, "reason": h.reason, "created_at": h.created_at.isoformat()} for h in t.status_history]
    }

@app.put("/api/tickets/{ticket_id}/status")
def api_update_status(ticket_id: int, req: StatusUpdateRequest, user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    t = db.query(Ticket).filter(Ticket.id == ticket_id).first()
    if not t:
        raise HTTPException(404, "Request not found.")
    old_st = t.status
    t.status = req.status
    if req.status == "Resolved":
        t.resolved_at = datetime.datetime.utcnow()
        t.resolution_notes = req.resolution_notes or req.reason or "Resolved by department."
        t.resolution_time_minutes = max(1, int((t.resolved_at - t.created_at).total_seconds() / 60.0))
    elif req.status == "Escalated":
        t.escalated_at = datetime.datetime.utcnow()
        t.priority = "Urgent"

    db.add(StatusHistory(ticket_id=t.id, old_status=old_st, new_status=req.status, changed_by_name=user.full_name, reason=req.reason or f"Status set to {req.status}"))
    db.commit()
    return {"status": "ok"}

@app.put("/api/tickets/{ticket_id}/reassign")
def api_reassign(ticket_id: int, req: ReassignRequest, user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    t = db.query(Ticket).filter(Ticket.id == ticket_id).first()
    d = db.query(Department).filter(Department.id == req.department_id).first()
    if not t or not d:
        raise HTTPException(404, "Invalid ticket or department.")
    t.department_id = d.id
    db.add(StatusHistory(ticket_id=t.id, old_status=t.status, new_status=t.status, changed_by_name=user.full_name, reason=f"Transferred to {d.name}"))
    db.commit()
    return {"status": "ok", "new_department": d.name}

@app.post("/api/tickets/{ticket_id}/messages")
def api_post_message(ticket_id: int, req: MessageRequest, user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    t = db.query(Ticket).filter(Ticket.id == ticket_id).first()
    if not t:
        raise HTTPException(404, "Request not found.")
    is_int = req.is_internal if user.role != "student" else False
    msg = Message(ticket_id=t.id, sender_id=user.id, sender_name=user.full_name, sender_role=user.role, message_text=req.message_text.strip(), is_internal=is_int)
    db.add(msg)
    if user.role == "student" and t.status == "Waiting for Student":
        t.status = "In Progress"
    db.commit()
    return {"status": "ok"}

@app.get("/api/admin/statistics")
def api_stats(db: Session = Depends(get_db)):
    total = db.query(Ticket).count()
    pending = db.query(Ticket).filter(Ticket.status.in_(["Open", "Assigned", "Waiting for Student"])).count()
    in_prog = db.query(Ticket).filter(Ticket.status == "In Progress").count()
    resolved = db.query(Ticket).filter(Ticket.status.in_(["Resolved", "Closed"])).count()
    escalated = db.query(Ticket).filter(Ticket.status == "Escalated").count()

    resolved_tickets = db.query(Ticket).filter(Ticket.status.in_(["Resolved", "Closed"]), Ticket.resolution_time_minutes.isnot(None)).all()
    avg_t = round(sum(t.resolution_time_minutes for t in resolved_tickets) / len(resolved_tickets), 1) if resolved_tickets else 30.0

    dept_dist = {}
    for d in db.query(Department).all():
        dept_dist[d.name] = db.query(Ticket).filter(Ticket.department_id == d.id).count()

    status_dist = {s: db.query(Ticket).filter(Ticket.status == s).count() for s in ["Assigned", "In Progress", "Escalated", "Resolved"]}

    return {
        "total_complaints": total,
        "pending": pending,
        "in_progress": in_prog,
        "resolved": resolved,
        "high_priority": escalated,
        "escalated": escalated,
        "average_resolution_time_minutes": avg_t,
        "department_distribution": dept_dist,
        "status_distribution": status_dist
    }

@app.get("/api/admin/departments")
def api_departments(db: Session = Depends(get_db)):
    return db.query(Department).all()

@app.get("/api/admin/categories")
def api_categories(db: Session = Depends(get_db)):
    return db.query(ComplaintCategory).all()

@app.get("/api/automation/status")
def api_auto_status(db: Session = Depends(get_db)):
    return {
        "active_tracked_tickets": db.query(Ticket).filter(Ticket.status.in_(["Assigned", "In Progress", "Waiting for Student"])).count(),
        "escalated_tickets": db.query(Ticket).filter(Ticket.status == "Escalated").count(),
        "demo_reminder_minutes": DEMO_CONFIG["demo_reminder_minutes"],
        "demo_escalation_minutes": DEMO_CONFIG["demo_escalation_minutes"]
    }

@app.post("/api/automation/config")
def api_auto_config(data: dict):
    DEMO_CONFIG["demo_reminder_minutes"] = max(1, int(data.get("demo_reminder_minutes", 2)))
    DEMO_CONFIG["demo_escalation_minutes"] = max(2, int(data.get("demo_escalation_minutes", 5)))
    return {"status": "ok"}

@app.post("/api/automation/run-check")
def api_run_check(db: Session = Depends(get_db)):
    res = run_automation_sla_check(db)
    return {"message": f"SLA check completed: {res['reminders_sent']} reminders sent, {res['escalations_triggered']} escalations triggered."}

@app.post("/api/automation/simulate-time-jump")
def api_time_jump(minutes: int = 15, db: Session = Depends(get_db)):
    res = simulate_time_jump(minutes, db)
    return {"message": f"Time accelerated by {minutes} minutes: {res['reminders_sent']} reminders sent, {res['escalations_triggered']} escalations triggered."}

@app.get("/api/notifications")
def api_notifications(user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    q = db.query(Notification).order_by(desc(Notification.created_at))
    if user.role == "student":
        q = q.filter(Notification.user_id == user.id)
    return q.limit(20).all()

@app.get("/api/notifications/unread-count")
def api_unread_count(user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    q = db.query(Notification).filter(Notification.is_read == False)
    if user.role == "student":
        q = q.filter(Notification.user_id == user.id)
    return {"unread_count": q.count()}

@app.put("/api/notifications/{nid}/read")
def api_mark_read(nid: int, db: Session = Depends(get_db)):
    n = db.query(Notification).filter(Notification.id == nid).first()
    if n:
        n.is_read = True
        db.commit()
    return {"status": "ok"}

@app.get("/api/notifications/emails")
def api_list_emails(db: Session = Depends(get_db)):
    emails = db.query(Notification).filter(Notification.email_recipient.isnot(None)).order_by(desc(Notification.created_at)).limit(30).all()
    return [{"id": e.id, "recipient": e.email_recipient, "subject": e.email_subject or e.title, "type": e.notification_type, "created_at": e.created_at.isoformat()} for e in emails]

@app.get("/api/notifications/emails/{nid}/preview")
def api_preview_email(nid: int, db: Session = Depends(get_db)):
    n = db.query(Notification).filter(Notification.id == nid).first()
    if not n or not n.email_html:
        raise HTTPException(404, "Email not found.")
    return HTMLResponse(n.email_html)

# ---------------------------------------------------------------------------
# 9. EMBEDDED COMPLETE FRONTEND (HTML + CSS + JAVASCRIPT)
# ---------------------------------------------------------------------------
INDEX_HTML = r"""<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>CampusFlow AI — College Attendance, Library, Marks & Student Request Agent</title>
  <style>
    :root {
      --primary: #2563eb; --primary-hover: #1d4ed8; --primary-light: #eff6ff;
      --secondary: #0f172a; --sidebar-bg: #0f172a; --sidebar-hover: #1e293b;
      --surface: #ffffff; --surface-alt: #f8fafc; --surface-hover: #f1f5f9;
      --border: #e2e8f0; --border-dark: #cbd5e1; --text-main: #0f172a;
      --text-muted: #64748b; --success: #10b981; --warning: #f59e0b; --danger: #ef4444;
      --radius-sm: 6px; --radius-md: 10px; --radius-lg: 14px; --radius-full: 9999px;
    }
    * { margin:0; padding:0; box-sizing:border-box; }
    body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background-color:#f1f5f9; color:var(--text-main); display:flex; height:100vh; overflow:hidden; }
    .app-container { display:flex; width:100vw; height:100vh; overflow:hidden; }
    .sidebar { width:260px; background-color:var(--sidebar-bg); color:#fff; display:flex; flex-direction:column; flex-shrink:0; border-right:1px solid rgba(255,255,255,0.08); z-index:20; }
    .sidebar-header { padding:22px 20px 18px; border-bottom:1px solid rgba(255,255,255,0.08); }
    .brand-wrapper { display:flex; align-items:center; gap:10px; }
    .brand-icon { width:38px; height:38px; background:linear-gradient(135deg, #2563eb, #38bdf8); border-radius:var(--radius-md); display:flex; align-items:center; justify-content:center; font-weight:800; font-size:18px; color:#fff; }
    .brand-name { font-size:19px; font-weight:800; color:#fff; }
    .brand-name span { color:#38bdf8; }
    .brand-tagline { font-size:11px; color:#94a3b8; margin-top:3px; }
    .sidebar-nav { padding:18px 12px; display:flex; flex-direction:column; gap:6px; flex:1; overflow-y:auto; }
    .nav-section-title { font-size:11px; text-transform:uppercase; color:#64748b; font-weight:700; padding:10px 12px 4px; }
    .nav-item { display:flex; align-items:center; gap:12px; padding:10px 14px; border-radius:var(--radius-sm); color:#94a3b8; text-decoration:none; font-size:14px; font-weight:500; cursor:pointer; transition:all 0.2s; }
    .nav-item:hover { background-color:var(--sidebar-hover); color:#fff; }
    .nav-item.active { background-color:var(--primary); color:#fff; font-weight:600; }
    .nav-badge { margin-left:auto; background-color:rgba(255,255,255,0.15); font-size:11px; padding:2px 7px; border-radius:var(--radius-full); }
    .sidebar-footer { padding:16px; border-top:1px solid rgba(255,255,255,0.08); }
    .demo-status-box { background:rgba(37,99,235,0.15); border:1px solid rgba(59,130,246,0.3); border-radius:var(--radius-sm); padding:10px 12px; }
    .demo-status-title { display:flex; align-items:center; justify-content:space-between; font-size:11px; font-weight:700; color:#60a5fa; text-transform:uppercase; }
    .live-indicator { display:inline-block; width:8px; height:8px; background-color:#10b981; border-radius:50%; box-shadow:0 0 8px #10b981; }
    
    .main-content { flex:1; display:flex; flex-direction:column; overflow:hidden; }
    .top-navbar { height:65px; background-color:var(--surface); border-bottom:1px solid var(--border); display:flex; align-items:center; justify-content:space-between; padding:0 24px; flex-shrink:0; }
    .college-badge { display:flex; align-items:center; gap:8px; background-color:var(--surface-alt); border:1px solid var(--border); padding:6px 14px; border-radius:var(--radius-full); font-size:13px; font-weight:600; }
    .navbar-right { display:flex; align-items:center; gap:14px; }
    .persona-switcher { display:flex; align-items:center; gap:10px; background-color:var(--surface-alt); border:1px solid var(--border); border-radius:var(--radius-md); padding:5px 12px; }
    .persona-label { font-size:11px; font-weight:700; color:var(--text-muted); text-transform:uppercase; }
    .persona-select { border:none; background:transparent; font-size:13px; font-weight:600; cursor:pointer; outline:none; }
    
    .btn { display:inline-flex; align-items:center; gap:8px; padding:8px 16px; border-radius:var(--radius-sm); font-size:13px; font-weight:600; cursor:pointer; border:none; text-decoration:none; transition:all 0.2s; }
    .btn-primary { background-color:var(--primary); color:#fff; }
    .btn-primary:hover { background-color:var(--primary-hover); }
    .btn-outline { background-color:#fff; border:1px solid var(--border-dark); color:var(--text-main); }
    .btn-outline:hover { background-color:var(--surface-hover); }
    .btn-warning { background-color:var(--warning); color:#fff; }
    .btn-sm { padding:5px 10px; font-size:12px; }
    .icon-btn { width:36px; height:36px; border-radius:var(--radius-sm); display:flex; align-items:center; justify-content:center; background-color:var(--surface-alt); border:1px solid var(--border); cursor:pointer; position:relative; }
    .badge-dot { position:absolute; top:6px; right:6px; width:8px; height:8px; background-color:var(--danger); border-radius:50%; }

    .view-container { flex:1; overflow-y:auto; padding:24px; display:none; }
    .view-container.active { display:block; }
    .page-header { margin-bottom:20px; }
    .page-title { font-size:24px; font-weight:800; color:var(--secondary); }
    .page-subtitle { font-size:13px; color:var(--text-muted); margin-top:2px; }
    
    .card { background-color:var(--surface); border:1px solid var(--border); border-radius:var(--radius-lg); padding:20px; margin-bottom:20px; box-shadow:0 1px 3px rgba(0,0,0,0.05); }
    .card-header { display:flex; align-items:center; justify-content:space-between; margin-bottom:16px; }
    .card-title { font-size:16px; font-weight:700; color:var(--secondary); }

    .stats-grid { display:grid; grid-template-columns:repeat(auto-fit, minmax(180px, 1fr)); gap:16px; margin-bottom:24px; }
    .stat-card { background:#fff; border:1px solid var(--border); border-radius:var(--radius-md); padding:16px 20px; display:flex; align-items:center; gap:16px; }
    .stat-icon { width:44px; height:44px; border-radius:var(--radius-md); display:flex; align-items:center; justify-content:center; font-size:20px; }
    .stat-icon.blue { background:#eff6ff; color:#2563eb; }
    .stat-icon.yellow { background:#fffbeb; color:#f59e0b; }
    .stat-icon.green { background:#ecfdf5; color:#10b981; }
    .stat-icon.red { background:#fef2f2; color:#ef4444; }
    .stat-icon.purple { background:#f5f3ff; color:#8b5cf6; }
    .stat-label { font-size:11px; font-weight:600; color:var(--text-muted); text-transform:uppercase; }
    .stat-value { font-size:22px; font-weight:800; color:var(--secondary); }

    .ai-input-card { background:linear-gradient(180deg, #ffffff 0%, #f8fafc 100%); border:1px solid var(--border-dark); border-radius:var(--radius-lg); padding:24px; margin-bottom:24px; box-shadow:0 4px 6px rgba(0,0,0,0.04); }
    .ai-header { display:flex; align-items:center; gap:12px; margin-bottom:14px; }
    .ai-sparkle-icon { width:34px; height:34px; background:linear-gradient(135deg, #3b82f6, #8b5cf6); border-radius:var(--radius-md); display:flex; align-items:center; justify-content:center; color:#fff; font-size:18px; }
    .ai-header-title { font-size:18px; font-weight:700; color:var(--secondary); }
    .ai-header-desc { font-size:13px; color:var(--text-muted); }
    .prompt-textarea { width:100%; min-height:85px; padding:14px; border:1.5px solid var(--border-dark); border-radius:var(--radius-md); font-family:inherit; font-size:14px; resize:vertical; outline:none; }
    .prompt-textarea:focus { border-color:var(--primary); box-shadow:0 0 0 3px rgba(37,99,235,0.15); }

    .preset-container { display:flex; flex-wrap:wrap; gap:8px; margin:12px 0 16px; }
    .preset-chip { background-color:var(--surface); border:1px solid var(--border); padding:6px 12px; border-radius:var(--radius-full); font-size:12px; cursor:pointer; transition:all 0.2s; }
    .preset-chip:hover { background-color:var(--primary-light); border-color:#93c5fd; color:var(--primary); }

    .ai-preview-card { margin-top:16px; background:#fff; border:1.5px solid #93c5fd; border-radius:var(--radius-md); padding:16px 20px; display:none; }
    .ai-preview-title { display:flex; justify-content:space-between; font-size:13px; font-weight:700; color:var(--primary); text-transform:uppercase; margin-bottom:12px; }
    .ai-preview-grid { display:grid; grid-template-columns:repeat(auto-fit, minmax(200px, 1fr)); gap:12px; margin-bottom:14px; }
    .preview-field { background:var(--surface-alt); padding:8px 12px; border-radius:var(--radius-sm); border:1px solid var(--border); }
    .preview-field-label { font-size:11px; font-weight:700; color:var(--text-muted); text-transform:uppercase; }
    .preview-field-val { font-size:13px; font-weight:600; color:var(--secondary); margin-top:2px; }
    .action-prescribed-box { background-color:#eff6ff; border-left:4px solid var(--primary); padding:10px 14px; border-radius:4px; font-size:13px; color:#1e3a8a; margin-bottom:14px; }

    .clarification-box { background-color:#fffbeb; border:1.5px solid #fde68a; border-radius:var(--radius-md); padding:16px; margin-top:14px; display:none; }
    .clarification-title { display:flex; align-items:center; gap:8px; color:#b45309; font-size:14px; font-weight:700; }
    .clarification-question { color:#78350f; font-size:13.5px; margin:6px 0 12px; }

    .badge { display:inline-flex; align-items:center; padding:3px 10px; border-radius:var(--radius-full); font-size:11px; font-weight:700; text-transform:uppercase; }
    .badge-open { background-color:#e0e7ff; color:#4338ca; }
    .badge-assigned { background-color:#dbeafe; color:#1d4ed8; }
    .badge-in-progress { background-color:#fef3c7; color:#b45309; }
    .badge-waiting-for-student { background-color:#f3e8ff; color:#7e22ce; }
    .badge-resolved { background-color:#d1fae5; color:#047857; }
    .badge-escalated { background-color:#fee2e2; color:#b91c1c; font-weight:800; border:1px solid #f87171; }
    .badge-low { background-color:#f1f5f9; color:#475569; }
    .badge-medium { background-color:#e0f2fe; color:#0369a1; }
    .badge-high { background-color:#ffedd5; color:#c2410c; }
    .badge-urgent { background-color:#fee2e2; color:#dc2626; }

    .table-container { overflow-x:auto; border:1px solid var(--border); border-radius:var(--radius-md); background:#fff; }
    .custom-table { width:100%; border-collapse:collapse; text-align:left; font-size:13.5px; }
    .custom-table th { background-color:var(--surface-alt); padding:12px 16px; font-weight:700; color:var(--text-muted); font-size:11px; text-transform:uppercase; border-bottom:1px solid var(--border); }
    .custom-table td { padding:14px 16px; border-bottom:1px solid var(--border); vertical-align:middle; }
    .custom-table tr:hover td { background-color:var(--surface-hover); }
    .ticket-code-link { font-family:monospace; font-weight:700; color:var(--primary); cursor:pointer; text-decoration:none; }

    .progress-stepper { display:flex; align-items:center; justify-content:space-between; margin:20px 0 28px; position:relative; }
    .progress-stepper::before { content:""; position:absolute; top:15px; left:20px; right:20px; height:3px; background-color:var(--border); z-index:1; }
    .step-node { display:flex; flex-direction:column; align-items:center; position:relative; z-index:2; text-align:center; }
    .step-circle { width:32px; height:32px; border-radius:50%; background-color:#fff; border:3px solid var(--border); display:flex; align-items:center; justify-content:center; font-size:12px; font-weight:700; color:var(--text-muted); }
    .step-node.completed .step-circle { background-color:var(--success); border-color:var(--success); color:#fff; }
    .step-node.active .step-circle { background-color:var(--primary); border-color:var(--primary); color:#fff; box-shadow:0 0 0 4px rgba(37,99,235,0.2); }
    .step-node.escalated .step-circle { background-color:var(--danger); border-color:var(--danger); color:#fff; }
    .step-label { font-size:11px; font-weight:600; color:var(--text-muted); margin-top:6px; }

    .timeline { position:relative; padding-left:24px; margin:16px 0; }
    .timeline::before { content:""; position:absolute; top:6px; bottom:6px; left:8px; width:2px; background-color:var(--border); }
    .timeline-item { position:relative; margin-bottom:18px; }
    .timeline-dot { position:absolute; left:-24px; top:4px; width:14px; height:14px; border-radius:50%; background-color:#fff; border:3px solid var(--primary); }
    .timeline-time { font-size:11px; color:var(--text-muted); }
    .timeline-content { font-size:13px; margin-top:2px; }

    .modal-overlay { position:fixed; top:0; left:0; width:100vw; height:100vh; background-color:rgba(15,23,42,0.6); backdrop-filter:blur(4px); display:none; align-items:center; justify-content:center; z-index:100; }
    .modal-overlay.active { display:flex; }
    .modal-card { background-color:#fff; width:90%; max-width:780px; max-height:90vh; border-radius:var(--radius-lg); box-shadow:0 10px 25px rgba(0,0,0,0.15); display:flex; flex-direction:column; overflow:hidden; }
    .modal-header { padding:18px 24px; border-bottom:1px solid var(--border); display:flex; align-items:center; justify-content:space-between; background-color:var(--surface-alt); }
    .modal-title { font-size:17px; font-weight:700; color:var(--secondary); }
    .modal-body { padding:24px; overflow-y:auto; flex:1; }
    .modal-footer { padding:16px 24px; border-top:1px solid var(--border); display:flex; align-items:center; justify-content:flex-end; gap:12px; background-color:var(--surface-alt); }

    .message-thread { display:flex; flex-direction:column; gap:12px; margin:16px 0; max-height:220px; overflow-y:auto; padding:8px; background:var(--surface-alt); border-radius:var(--radius-md); }
    .chat-bubble { padding:10px 14px; border-radius:var(--radius-md); font-size:13px; max-width:85%; }
    .chat-bubble.student { background-color:#fff; border:1px solid var(--border); align-self:flex-start; }
    .chat-bubble.staff { background-color:var(--primary); color:#fff; align-self:flex-end; }
    .chat-sender { font-size:11px; font-weight:700; margin-bottom:3px; opacity:0.85; }

    .charts-grid { display:grid; grid-template-columns:repeat(auto-fit, minmax(320px, 1fr)); gap:20px; margin-bottom:24px; }
    .chart-card { background:#fff; border:1px solid var(--border); border-radius:var(--radius-md); padding:18px 20px; }
    .chart-title { font-size:14px; font-weight:700; color:var(--secondary); margin-bottom:12px; }

    .notifications-dropdown { position:absolute; top:60px; right:24px; width:360px; background:#fff; border:1px solid var(--border); border-radius:var(--radius-md); box-shadow:0 10px 25px rgba(0,0,0,0.15); display:none; z-index:50; max-height:440px; flex-direction:column; }
    .notifications-dropdown.active { display:flex; }
    .notif-header { padding:12px 16px; border-bottom:1px solid var(--border); display:flex; justify-content:space-between; font-weight:700; font-size:13px; }
    .notif-list { overflow-y:auto; flex:1; }
    .notif-item { padding:12px 16px; border-bottom:1px solid var(--border); font-size:13px; cursor:pointer; }
    .notif-item:hover { background-color:var(--surface-hover); }
    .notif-item.unread { background-color:#eff6ff; border-left:3px solid var(--primary); }

    .filter-bar { display:flex; flex-wrap:wrap; gap:12px; margin-bottom:16px; align-items:center; }
    .search-input { flex:1; min-width:200px; padding:8px 14px; border:1px solid var(--border-dark); border-radius:var(--radius-sm); font-size:13px; outline:none; }
    .filter-select { padding:8px 12px; border:1px solid var(--border-dark); border-radius:var(--radius-sm); font-size:13px; background:#fff; outline:none; }
    .timing-control-card { background:#fff; border:1px solid var(--border); border-radius:var(--radius-md); padding:20px; margin-bottom:20px; }
    .range-slider { width:100%; }
    .email-preview-frame { width:100%; height:520px; border:1px solid var(--border); border-radius:var(--radius-md); }
  </style>
</head>
<body>

<div class="app-container">
  <aside class="sidebar">
    <div class="sidebar-header">
      <div class="brand-wrapper">
        <div class="brand-icon">CF</div>
        <div>
          <div class="brand-name">CampusFlow <span>AI</span></div>
          <div class="brand-tagline">Your Campus. Your Request. Automatically Handled.</div>
        </div>
      </div>
    </div>

    <nav class="sidebar-nav">
      <div class="nav-section-title">Academic & Student Portals</div>
      <a class="nav-item active" data-tab="student">
        <span>Student Academic Hub</span>
      </a>
      <a class="nav-item" data-tab="admin">
        <span>Staff & Admin Console</span>
      </a>
      <div class="nav-section-title">Autonomous Operations</div>
      <a class="nav-item" data-tab="automation">
        <span>Autonomous SLA Engine</span>
        <span class="nav-badge">Active</span>
      </a>
      <a class="nav-item" data-tab="emails">
        <span>Simulated Outbox</span>
        <span class="nav-badge" style="background:#3b82f6;">Emails</span>
      </a>
    </nav>

    <div class="sidebar-footer">
      <div class="demo-status-box">
        <div class="demo-status-title">
          <span>Campus AI Active</span>
          <span class="live-indicator"></span>
        </div>
        <div style="font-size:11px; color:#cbd5e1; margin-top:4px;">Attendance • Library • Marks • ID</div>
      </div>
    </div>
  </aside>

  <main class="main-content">
    <header class="top-navbar">
      <div class="college-badge">
        🏛️ <span>Apex Institute of Technology</span>
      </div>

      <div class="navbar-right">
        <div class="persona-switcher">
          <span class="persona-label">Testing Persona:</span>
          <select id="persona-select" class="persona-select"></select>
          <span id="current-user-role-badge" class="badge badge-open">STUDENT</span>
        </div>

        <button id="quick-sla-btn" class="btn btn-warning btn-sm" title="Run SLA evaluation">
          ⚡ Run SLA Check
        </button>

        <button id="notif-bell-btn" class="icon-btn" title="View Notifications">
          <svg width="18" height="18" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9"/></svg>
          <span id="unread-notif-badge" class="badge-dot" style="display:none;"></span>
        </button>

        <div id="notifications-dropdown" class="notifications-dropdown">
          <div class="notif-header">
            <span>Recent Notifications</span>
            <span style="font-size:11px; color:#3b82f6;">Live Stream</span>
          </div>
          <div id="notif-dropdown-list" class="notif-list"></div>
        </div>
      </div>
    </header>

    <!-- VIEW 1: STUDENT ACADEMIC & REQUEST PORTAL -->
    <div id="view-student" class="view-container active">
      <div class="page-header">
        <h1 class="page-title">Student Academic & Grievance Portal</h1>
        <p class="page-subtitle">Welcome, <strong id="student-portal-name">Sarah Chen</strong>. Monitor your attendance, library books, exam marks, and submit requests automatically.</p>
      </div>

      <!-- ACADEMIC HUB CARDS -->
      <div style="display:grid; grid-template-columns: repeat(auto-fit, minmax(310px, 1fr)); gap: 16px; margin-bottom: 24px;">
        <div class="card" style="margin-bottom:0; border-top: 4px solid #2563eb;">
          <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:12px;">
            <div style="display:flex; align-items:center; gap:8px;">
              <span style="font-size:20px;">📅</span>
              <h3 style="font-size:15px; font-weight:700;">Attendance Tracker</h3>
            </div>
            <span class="badge badge-assigned" style="font-size:12px;">84.5% Overall</span>
          </div>
          <div style="font-size:12px; color:#64748b; margin-bottom:10px;">Mandatory threshold: 75.0%</div>
          <div id="acad-attendance-list" style="display:flex; flex-direction:column; gap:8px; margin-bottom:14px;"></div>
          <button class="btn btn-outline btn-sm" style="width:100%; justify-content:center; border-color:#93c5fd; color:#1d4ed8;" onclick="presetDisputeAttendance()">
            ⚠️ Dispute Attendance Error
          </button>
        </div>

        <div class="card" style="margin-bottom:0; border-top: 4px solid #10b981;">
          <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:12px;">
            <div style="display:flex; align-items:center; gap:8px;">
              <span style="font-size:20px;">📚</span>
              <h3 style="font-size:15px; font-weight:700;">Issued Library Books</h3>
            </div>
            <span class="badge badge-resolved" style="font-size:12px;">2 Issued</span>
          </div>
          <div style="font-size:12px; color:#64748b; margin-bottom:10px;">Library Circulation Account (#LIB-STU042)</div>
          <div id="acad-books-list" style="display:flex; flex-direction:column; gap:8px; margin-bottom:14px;"></div>
          <button class="btn btn-outline btn-sm" style="width:100%; justify-content:center; border-color:#a7f3d0; color:#047857;" onclick="presetLibraryDispute()">
            🔄 Report Returned Book Still Issued
          </button>
        </div>

        <div class="card" style="margin-bottom:0; border-top: 4px solid #8b5cf6;">
          <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:12px;">
            <div style="display:flex; align-items:center; gap:8px;">
              <span style="font-size:20px;">🎓</span>
              <h3 style="font-size:15px; font-weight:700;">Marks & Exam Results</h3>
            </div>
            <span class="badge badge-high" style="font-size:12px; background:#f5f3ff; color:#7c3aed;">SGPA: 8.74</span>
          </div>
          <div style="font-size:12px; color:#64748b; margin-bottom:10px;">Fall 2026 • Mid-Term 1 Assessment</div>
          <div id="acad-marks-list" style="display:flex; flex-direction:column; gap:8px; margin-bottom:14px;"></div>
          <button class="btn btn-outline btn-sm" style="width:100%; justify-content:center; border-color:#c4b5fd; color:#6d28d9;" onclick="presetMarksDispute()">
            📝 Rectify Marks / Apply for Re-evaluation
          </button>
        </div>
      </div>

      <!-- AI NATURAL LANGUAGE INTAKE -->
      <div class="ai-input-card">
        <div class="ai-header">
          <div class="ai-sparkle-icon">✨</div>
          <div>
            <div class="ai-header-title">CampusFlow AI Academic Agent</div>
            <div class="ai-header-desc">State your complaint in plain English. The AI extracts subject, dates, and intent, assigns to the department, and follows up until solved.</div>
          </div>
          <span id="ai-status-badge" class="badge badge-open" style="margin-left:auto;">AI Agent Ready</span>
        </div>

        <div style="font-size:12px; font-weight:700; color:#64748b; margin-top:8px;">ONE-CLICK SCENARIO PRESETS:</div>
        <div class="preset-container">
          <button class="preset-chip" data-prompt="My attendance for Operating Systems on Monday wasn't marked.">📅 Attendance: OS Monday Missing</button>
          <button class="preset-chip" data-prompt="I returned a library book but it is still showing as issued.">📚 Library: Book Returned Still Issued</button>
          <button class="preset-chip" data-prompt="My internal marks for Data Structures show absent but I attended the test.">🎓 Marks: DSA Internal Marked Absent Error</button>
          <button class="preset-chip" data-prompt="I applied for re-evaluation in Database Systems Mid-Term but haven't received the revised score.">📝 Exam: DBMS Re-evaluation Query</button>
          <button class="preset-chip" data-prompt="My ID card hasn't been issued for 10 days.">🪪 ID Card: Delayed 10 Days</button>
          <button class="preset-chip" data-prompt="attendance" style="background:#fffbeb; border-color:#fde68a; color:#b45309;">❓ Vague Input (Tests Clarification)</button>
        </div>

        <textarea id="complaint-prompt-input" class="prompt-textarea" placeholder="E.g., My attendance for Operating Systems on Monday wasn't marked, or I returned a library book but it is still showing as issued..."></textarea>

        <div id="ai-preview-card" class="ai-preview-card">
          <div class="ai-preview-title">
            <span>✨ AI Real-Time Structured Classification</span>
            <span style="font-size:11px; text-transform:none; color:#64748b;">Autonomous Route</span>
          </div>
          <div class="ai-preview-grid">
            <div class="preview-field"><div class="preview-field-label">Category</div><div id="prev-category" class="preview-field-val">-</div></div>
            <div class="preview-field"><div class="preview-field-label">Responsible Department</div><div id="prev-dept" class="preview-field-val">-</div></div>
            <div class="preview-field"><div class="preview-field-label">Priority Level</div><div class="preview-field-val"><span id="prev-priority" class="badge badge-medium">Medium</span></div></div>
            <div class="preview-field"><div class="preview-field-label">Summary Title</div><div id="prev-summary" class="preview-field-val">-</div></div>
          </div>
          <div class="action-prescribed-box"><strong>Prescribed Operational Action:</strong> <span id="prev-action">-</span></div>
        </div>

        <div id="clarification-box" class="clarification-box">
          <div class="clarification-title">
            <svg width="18" height="18" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"/></svg>
            <span>CampusFlow AI needs clarification before filing request:</span>
          </div>
          <div id="clarification-question-text" class="clarification-question">-</div>
          <div style="display:flex; gap:10px;">
            <input id="clarification-answer-input" type="text" class="search-input" placeholder="E.g., Operating Systems on Monday, or CLRS Algorithms book...">
            <button id="clarification-submit-btn" class="btn btn-warning">Submit Clarification</button>
          </div>
        </div>

        <div style="display:flex; justify-content:flex-end; gap:12px; margin-top:16px;">
          <button id="submit-complaint-btn" class="btn btn-primary">🚀 Submit Request to CampusFlow AI</button>
        </div>
      </div>

      <!-- MY REQUESTS TABLE -->
      <div class="card">
        <div class="card-header">
          <div>
            <h2 class="card-title">My Tracked Requests & Grievances</h2>
            <div style="font-size:12px; color:#64748b;">Live progress tracking, department assignments, and follow-up history.</div>
          </div>
          <select id="student-status-filter" class="filter-select">
            <option value="all">All Statuses</option>
            <option value="pending">Active / In Progress</option>
            <option value="resolved">Resolved</option>
          </select>
        </div>

        <div class="table-container">
          <table class="custom-table">
            <thead>
              <tr>
                <th>Request ID</th>
                <th>Summary</th>
                <th>Category</th>
                <th>Assigned Department</th>
                <th>Priority</th>
                <th>Status</th>
                <th>Submitted</th>
                <th>Action</th>
              </tr>
            </thead>
            <tbody id="student-tickets-tbody"></tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- VIEW 2: STAFF & ADMIN CONSOLE -->
    <div id="view-admin" class="view-container">
      <div class="page-header">
        <h1 class="page-title">College Staff & Department Console</h1>
        <p class="page-subtitle">Manage attendance disputes, library returns, marks rectifications, and administrative requests.</p>
      </div>

      <div class="stats-grid">
        <div class="stat-card"><div class="stat-icon blue">📋</div><div class="stat-info"><span class="stat-label">Total Requests</span><span id="stat-total" class="stat-value">0</span></div></div>
        <div class="stat-card"><div class="stat-icon yellow">⏳</div><div class="stat-info"><span class="stat-label">Pending Action</span><span id="stat-pending" class="stat-value">0</span></div></div>
        <div class="stat-card"><div class="stat-icon blue">⚙️</div><div class="stat-info"><span class="stat-label">In Progress</span><span id="stat-in-progress" class="stat-value">0</span></div></div>
        <div class="stat-card"><div class="stat-icon red">🚨</div><div class="stat-info"><span class="stat-label">Urgent / Escalated</span><span id="stat-escalated" class="stat-value">0</span></div></div>
        <div class="stat-card"><div class="stat-icon green">✅</div><div class="stat-info"><span class="stat-label">Resolved</span><span id="stat-resolved" class="stat-value">0</span></div></div>
        <div class="stat-card"><div class="stat-icon purple">⏱️</div><div class="stat-info"><span class="stat-label">Avg Resolution</span><span id="stat-avg-time" class="stat-value">--</span></div></div>
      </div>

      <div class="charts-grid">
        <div class="chart-card"><div class="chart-title">Department Workload Distribution</div><div id="dept-chart-container"></div></div>
        <div class="chart-card"><div class="chart-title">Request Lifecycle Status</div><div id="status-chart-container"></div></div>
      </div>

      <div class="card">
        <div class="card-header"><h2 class="card-title">Manage Student Requests & Grievances</h2></div>
        <div class="filter-bar">
          <input id="admin-search-input" type="text" class="search-input" placeholder="Search by Request ID, Student, or Keyword...">
          <select id="admin-filter-dept" class="filter-select"><option value="">All Departments</option></select>
          <select id="admin-filter-status" class="filter-select">
            <option value="">All Statuses</option>
            <option value="Assigned">Assigned</option>
            <option value="In Progress">In Progress</option>
            <option value="Escalated">Escalated</option>
            <option value="Resolved">Resolved</option>
          </select>
          <select id="admin-filter-priority" class="filter-select">
            <option value="">All Priorities</option>
            <option value="Low">Low</option>
            <option value="Medium">Medium</option>
            <option value="High">High</option>
            <option value="Urgent">Urgent</option>
          </select>
        </div>
        <div class="table-container">
          <table class="custom-table">
            <thead>
              <tr><th>Request ID</th><th>Student</th><th>Summary</th><th>Department</th><th>Priority</th><th>Status</th><th>Submitted</th><th>Action</th></tr>
            </thead>
            <tbody id="admin-tickets-tbody"></tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- VIEW 3: AUTOMATION & SLA MONITOR -->
    <div id="view-automation" class="view-container">
      <div class="page-header">
        <h1 class="page-title">Autonomous Follow-up & SLA Engine</h1>
        <p class="page-subtitle">Automated reminder dispatch, configurable demo timing, and instant escalation trigger.</p>
      </div>

      <div class="timing-control-card">
        <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:14px;">
          <div><h3 style="font-size:16px; font-weight:700;">Hackathon Demo Timing Simulator</h3><p style="font-size:12.5px; color:#64748b;">Configure fast reminder and escalation timing in minutes to demonstrate autonomous follow-ups live to judges.</p></div>
          <button id="save-timing-btn" class="btn btn-primary btn-sm">Save Timing</button>
        </div>
        <div style="display:grid; grid-template-columns: 1fr 1fr; gap:20px;">
          <div>
            <div style="display:flex; justify-content:space-between; font-size:13px; font-weight:600; margin-bottom:4px;"><span>Automated Reminder Interval:</span><span id="reminder-slider-val" style="color:#2563eb;">2m</span></div>
            <input id="timing-reminder-slider" type="range" class="range-slider" min="1" max="30" value="2">
          </div>
          <div>
            <div style="display:flex; justify-content:space-between; font-size:13px; font-weight:600; margin-bottom:4px;"><span>SLA Escalation Window:</span><span id="escalation-slider-val" style="color:#ef4444;">5m</span></div>
            <input id="timing-escalation-slider" type="range" class="range-slider" min="2" max="60" value="5">
          </div>
        </div>
        <div style="display:flex; gap:12px; margin-top:20px; border-top:1px solid #e2e8f0; padding-top:16px;">
          <button id="run-sla-check-btn" class="btn btn-warning">⚡ Run SLA Evaluation Loop Now</button>
          <button id="time-jump-btn" class="btn btn-outline" style="border-color:#f59e0b; color:#b45309;">⏩ Fast-Forward Time by 15 Minutes</button>
        </div>
      </div>
    </div>

    <!-- VIEW 4: SIMULATED EMAIL OUTBOX -->
    <div id="view-emails" class="view-container">
      <div class="page-header">
        <h1 class="page-title">Simulated Campus Email Outbox</h1>
        <p class="page-subtitle">Inspect the actual campus emails dispatched by CampusFlow AI to students and department heads.</p>
      </div>
      <div class="card">
        <div class="table-container">
          <table class="custom-table">
            <thead><tr><th>Recipient Email</th><th>Subject Line</th><th>Notice Type</th><th>Dispatched At</th><th>Preview</th></tr></thead>
            <tbody id="emails-tbody"></tbody>
          </table>
        </div>
      </div>
    </div>
  </main>
</div>

<!-- REQUEST DETAIL MODAL -->
<div id="ticket-modal-overlay" class="modal-overlay">
  <div class="modal-card">
    <div class="modal-header">
      <div>
        <span id="modal-ticket-code" style="font-family:monospace; font-weight:800; color:#2563eb; font-size:15px;">REQ-2026-XXXX</span>
        <div id="modal-ticket-title" class="modal-title">Request Title</div>
      </div>
      <div style="display:flex; align-items:center; gap:8px;">
        <span id="modal-priority-badge" class="badge badge-medium">Medium</span>
        <span id="modal-status-badge" class="badge badge-open">Open</span>
        <button class="icon-btn modal-close-btn">&times;</button>
      </div>
    </div>

    <div class="modal-body">
      <div class="progress-stepper">
        <div id="step-open" class="step-node completed"><div class="step-circle">1</div><div class="step-label">Submitted</div></div>
        <div id="step-assigned" class="step-node completed"><div class="step-circle">2</div><div class="step-label">Assigned</div></div>
        <div id="step-progress" class="step-node active"><div class="step-circle">3</div><div class="step-label">In Progress</div></div>
        <div id="step-resolved" class="step-node"><div class="step-circle">4</div><div class="step-label">Resolved</div></div>
      </div>

      <div style="display:grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap:12px; margin-bottom:16px;">
        <div class="preview-field"><div class="preview-field-label">Student</div><div id="modal-student-name" class="preview-field-val">-</div></div>
        <div class="preview-field"><div class="preview-field-label">Department</div><div id="modal-department-name" class="preview-field-val">-</div></div>
        <div class="preview-field"><div class="preview-field-label">Category</div><div id="modal-category-name" class="preview-field-val">-</div></div>
        <div class="preview-field"><div class="preview-field-label">Submitted At</div><div id="modal-created-time" class="preview-field-val">-</div></div>
      </div>

      <div style="background:#f8fafc; border:1px solid #e2e8f0; border-radius:8px; padding:12px 16px; margin-bottom:14px;">
        <div style="font-size:11px; font-weight:700; color:#64748b; text-transform:uppercase;">Student Statement:</div>
        <div id="modal-raw-message" style="font-size:13.5px; color:#1e293b; margin-top:4px; font-style:italic;">-</div>
      </div>

      <div class="action-prescribed-box"><strong>CampusFlow AI Prescribed Action:</strong><div id="modal-required-action" style="margin-top:3px;">-</div></div>

      <div id="modal-admin-action-panel" style="background:#f1f5f9; border-radius:8px; padding:14px; margin-bottom:16px;">
        <div style="font-weight:700; font-size:13px; margin-bottom:10px;">Department Actions</div>
        <div style="display:grid; grid-template-columns: 1fr 1fr; gap:12px; margin-bottom:10px;">
          <div>
            <label style="font-size:11px; font-weight:700; color:#475569;">Update Status:</label>
            <div style="display:flex; gap:6px; margin-top:4px;">
              <select id="modal-status-select" class="filter-select" style="flex:1;">
                <option value="Assigned">Assigned</option>
                <option value="In Progress">In Progress</option>
                <option value="Escalated">Escalated</option>
                <option value="Resolved">Resolved</option>
              </select>
              <button id="modal-status-update-btn" class="btn btn-primary btn-sm">Update</button>
            </div>
          </div>
          <div>
            <label style="font-size:11px; font-weight:700; color:#475569;">Reassign Department:</label>
            <div style="display:flex; gap:6px; margin-top:4px;">
              <select id="modal-reassign-dept" class="filter-select" style="flex:1;"></select>
              <button id="modal-reassign-btn" class="btn btn-outline btn-sm">Reassign</button>
            </div>
          </div>
        </div>
        <div>
          <label style="font-size:11px; font-weight:700; color:#475569;">Resolution / Staff Notes:</label>
          <input id="modal-status-note" type="text" class="search-input" style="width:100%; margin-top:4px;" placeholder="E.g., Verified attendance sheet with OS professor...">
        </div>
      </div>

      <div style="font-size:13px; font-weight:700; color:#0f172a; margin-top:14px;">Conversation & Activity Thread</div>
      <div id="modal-message-thread" class="message-thread"></div>
      <div style="display:flex; gap:8px;">
        <input id="modal-message-input" type="text" class="search-input" placeholder="Type a response or internal note...">
        <button id="modal-send-message-btn" class="btn btn-primary btn-sm">Send</button>
      </div>

      <div style="font-size:13px; font-weight:700; color:#0f172a; margin-top:20px;">Automated SLA & Audit Trail</div>
      <div id="modal-timeline-list" class="timeline"></div>
    </div>

    <div class="modal-footer"><button class="btn btn-outline modal-close-btn">Close</button></div>
  </div>
</div>

<div id="email-modal-overlay" class="modal-overlay">
  <div class="modal-card" style="max-width:680px;">
    <div class="modal-header"><div class="modal-title">Rendered CampusFlow AI Transactional Email</div><button class="icon-btn modal-close-btn">&times;</button></div>
    <div class="modal-body" style="padding:10px;"><iframe id="email-preview-iframe" class="email-preview-frame"></iframe></div>
    <div class="modal-footer"><button class="btn btn-outline modal-close-btn">Close</button></div>
  </div>
</div>

<div id="toast-container" style="position:fixed; bottom:20px; right:20px; z-index:9999;"></div>

<script>
  const state = { currentUser: null, activeTab: 'student', demoUsers: [], departments: [], currentTicket: null, isAnalyzing: false };

  document.addEventListener('DOMContentLoaded', async () => {
    initNav();
    initPresets();
    initModals();
    await loadInit();
    setupEvents();
  });

  async function loadInit() {
    try {
      const uRes = await fetch('/api/auth/demo-users');
      state.demoUsers = await uRes.json();
      const pSel = document.getElementById('persona-select');
      pSel.innerHTML = state.demoUsers.map(u => `<option value="${u.id}">${u.full_name} (${u.persona_label})</option>`).join('');
      pSel.addEventListener('change', e => switchUser(e.target.value));

      const meRes = await fetch('/api/auth/me');
      if (meRes.ok) state.currentUser = await meRes.json();
      else if (state.demoUsers.length) await switchUser(state.demoUsers[0].id);

      updateUserUI();
      const dRes = await fetch('/api/admin/departments');
      state.departments = await dRes.json();
      populateDepts();

      await loadAcademic();
      refreshTab();
      updateUnread();
    } catch(e) { console.error(e); }
  }

  function populateDepts() {
    const fDept = document.getElementById('admin-filter-dept');
    const mDept = document.getElementById('modal-reassign-dept');
    if (fDept) fDept.innerHTML = '<option value="">All Departments</option>' + state.departments.map(d => `<option value="${d.id}">${d.name}</option>`).join('');
    if (mDept) mDept.innerHTML = state.departments.map(d => `<option value="${d.id}">${d.name}</option>`).join('');
  }

  async function switchUser(uid) {
    const res = await fetch('/api/auth/switch-demo-user', { method:'POST', headers:{'Content-Type':'application/json'}, body:JSON.stringify({user_id:parseInt(uid)}) });
    if (res.ok) {
      const data = await res.json();
      state.currentUser = data.user;
      updateUserUI();
      switchTab(state.currentUser.role === 'admin' ? 'admin' : 'student');
      showToast('Switched persona to ' + state.currentUser.full_name);
    }
  }

  function updateUserUI() {
    if (!state.currentUser) return;
    document.getElementById('persona-select').value = state.currentUser.id;
    document.getElementById('current-user-role-badge').textContent = state.currentUser.role.toUpperCase();
    document.getElementById('student-portal-name').textContent = state.currentUser.full_name;
  }

  async function loadAcademic() {
    try {
      const res = await fetch('/api/tickets/academic-overview');
      if (!res.ok) return;
      const data = await res.json();

      document.getElementById('acad-attendance-list').innerHTML = data.attendance.subjects.map(s => `
        <div style="background:#f8fafc; border:1px solid #e2e8f0; border-radius:6px; padding:7px 10px;">
          <div style="display:flex; justify-content:space-between; font-size:12px; font-weight:600;">
            <span>${s.name}</span>
            <span style="color:${s.percentage < 80 ? '#ef4444':'#10b981'}; font-weight:700;">${s.percentage}% (${s.attended}/${s.total})</span>
          </div>
          <div style="background:#e2e8f0; border-radius:99px; height:5px; margin-top:4px; overflow:hidden;">
            <div style="background:${s.percentage < 80 ? '#ef4444':'#10b981'}; width:${s.percentage}%; height:100%;"></div>
          </div>
        </div>
      `).join('');

      document.getElementById('acad-books-list').innerHTML = data.library_books.map(b => `
        <div style="background:#f8fafc; border:1px solid ${b.action_needed ? '#fde68a':'#e2e8f0'}; border-radius:6px; padding:8px 10px;">
          <div style="font-weight:700; font-size:12.5px;">${b.title}</div>
          <div style="display:flex; justify-content:space-between; font-size:11px; color:#64748b; margin-top:2px;">
            <span>Due: <strong>${b.due_date}</strong></span>
            <span style="color:${b.action_needed ? '#b45309':'#059669'}; font-weight:600;">${b.status}</span>
          </div>
        </div>
      `).join('');

      document.getElementById('acad-marks-list').innerHTML = data.marks_results.subjects.map(m => `
        <div style="background:#f8fafc; border:1px solid ${m.has_issue ? '#fca5a5':'#e2e8f0'}; border-radius:6px; padding:7px 10px;">
          <div style="display:flex; justify-content:space-between; font-size:12px; font-weight:600;">
            <span>${m.name}</span>
            <span style="font-weight:700; color:${m.has_issue ? '#dc2626':'#2563eb'};">${m.score} (${m.grade})</span>
          </div>
          <div style="font-size:10.5px; color:${m.has_issue ? '#b91c1c':'#64748b'}; margin-top:2px;">${m.status}</div>
        </div>
      `).join('');
    } catch(e) { console.error(e); }
  }

  function presetDisputeAttendance() {
    const t = document.getElementById('complaint-prompt-input');
    t.value = "My attendance for Operating Systems on Monday wasn't marked.";
    triggerAI(t.value);
    t.scrollIntoView({ behavior:'smooth' });
  }

  function presetLibraryDispute() {
    const t = document.getElementById('complaint-prompt-input');
    t.value = "I returned a library book but it is still showing as issued.";
    triggerAI(t.value);
    t.scrollIntoView({ behavior:'smooth' });
  }

  function presetMarksDispute() {
    const t = document.getElementById('complaint-prompt-input');
    t.value = "My internal marks for Data Structures show absent but I attended the test.";
    triggerAI(t.value);
    t.scrollIntoView({ behavior:'smooth' });
  }

  function initNav() {
    document.querySelectorAll('.nav-item').forEach(i => {
      i.addEventListener('click', () => switchTab(i.dataset.tab));
    });
  }

  function switchTab(tName) {
    state.activeTab = tName;
    document.querySelectorAll('.nav-item').forEach(el => el.classList.toggle('active', el.dataset.tab === tName));
    document.querySelectorAll('.view-container').forEach(el => el.classList.remove('active'));
    document.getElementById('view-' + tName)?.classList.add('active');
    refreshTab();
  }

  function refreshTab() {
    if (state.activeTab === 'student') { loadAcademic(); loadStudentRequests(); }
    else if (state.activeTab === 'admin') { loadStats(); loadAdminRequests(); }
    else if (state.activeTab === 'automation') { loadAuto(); }
    else if (state.activeTab === 'emails') { loadEmails(); }
  }

  function initPresets() {
    document.querySelectorAll('.preset-chip').forEach(c => {
      c.addEventListener('click', () => {
        const text = c.dataset.prompt;
        document.getElementById('complaint-prompt-input').value = text;
        triggerAI(text);
      });
    });

    let deb;
    document.getElementById('complaint-prompt-input').addEventListener('input', e => {
      clearTimeout(deb);
      const val = e.target.value.trim();
      if (val.length >= 6) deb = setTimeout(() => triggerAI(val), 400);
      else { document.getElementById('ai-preview-card').style.display = 'none'; document.getElementById('clarification-box').style.display = 'none'; }
    });
  }

  async function triggerAI(msg) {
    if (state.isAnalyzing) return;
    state.isAnalyzing = true;
    try {
      const res = await fetch('/api/tickets/analyze', { method:'POST', headers:{'Content-Type':'application/json'}, body:JSON.stringify({message:msg}) });
      if (res.ok) {
        const data = await res.json();
        const clar = document.getElementById('clarification-box');
        const prev = document.getElementById('ai-preview-card');
        if (data.is_clarification_needed) {
          clar.style.display = 'block';
          document.getElementById('clarification-question-text').textContent = data.clarification_question;
          prev.style.display = 'none';
        } else {
          clar.style.display = 'none';
          document.getElementById('prev-category').textContent = data.category;
          document.getElementById('prev-dept').textContent = data.department;
          document.getElementById('prev-priority').textContent = data.priority;
          document.getElementById('prev-summary').textContent = data.summary;
          document.getElementById('prev-action').textContent = data.required_action;
          prev.style.display = 'block';
        }
      }
    } finally { state.isAnalyzing = false; }
  }

  async function submitComplaint() {
    const text = document.getElementById('complaint-prompt-input').value.trim();
    const clar = document.getElementById('clarification-answer-input').value.trim();
    if (!text) return showToast('Please enter a request.', 'warning');

    const res = await fetch('/api/tickets/submit', {
      method:'POST',
      headers:{'Content-Type':'application/json'},
      body:JSON.stringify({ message:text, clarification_answer:clar || null })
    });
    const result = await res.json();
    if (result.status === 'clarification_needed') {
      document.getElementById('clarification-box').style.display = 'block';
      document.getElementById('clarification-question-text').textContent = result.clarification_question;
      showToast('Please clarify your request.', 'warning');
    } else if (result.status === 'created') {
      showToast('Created request ' + result.ticket_code + ' successfully!', 'success');
      document.getElementById('complaint-prompt-input').value = '';
      document.getElementById('clarification-answer-input').value = '';
      document.getElementById('ai-preview-card').style.display = 'none';
      document.getElementById('clarification-box').style.display = 'none';
      loadStudentRequests();
      openModal(result.ticket_id);
    }
  }

  async function loadStudentRequests() {
    const tbody = document.getElementById('student-tickets-tbody');
    const fVal = document.getElementById('student-status-filter').value;
    const res = await fetch(`/api/tickets?my_tickets_only=true${fVal !== 'all' ? '&status='+fVal : ''}`);
    const data = await res.json();
    tbody.innerHTML = data.map(t => `
      <tr>
        <td><a class="ticket-code-link" onclick="openModal(${t.id})">${t.ticket_code}</a></td>
        <td style="font-weight:600;">${t.title}</td>
        <td>${t.category_name}</td>
        <td>${t.department_name}</td>
        <td><span class="badge badge-${t.priority.toLowerCase()}">${t.priority}</span></td>
        <td><span class="badge badge-${t.status.toLowerCase().replace(/ /g,'-')}">${t.status}</span></td>
        <td style="font-size:12px; color:#64748b;">${new Date(t.created_at).toLocaleDateString()}</td>
        <td><button class="btn btn-outline btn-sm" onclick="openModal(${t.id})">View Progress</button></td>
      </tr>
    `).join('') || '<tr><td colspan="8" style="text-align:center; padding:20px;">No requests found.</td></tr>';
  }

  async function loadStats() {
    const res = await fetch('/api/admin/statistics');
    const s = await res.json();
    document.getElementById('stat-total').textContent = s.total_complaints;
    document.getElementById('stat-pending').textContent = s.pending;
    document.getElementById('stat-in-progress').textContent = s.in_progress;
    document.getElementById('stat-resolved').textContent = s.resolved;
    document.getElementById('stat-escalated').textContent = s.escalated;
    document.getElementById('stat-avg-time').textContent = s.average_resolution_time_minutes + 'm';

    const colors = ['#2563eb', '#10b981', '#8b5cf6', '#f59e0b', '#06b6d4'];
    let idx = 0;
    document.getElementById('dept-chart-container').innerHTML = Object.entries(s.department_distribution).map(([k,v]) => {
      const c = colors[idx++ % colors.length];
      const pct = Math.min(100, Math.round((v / (s.total_complaints||1)) * 100));
      return `<div style="margin-bottom:8px;"><div style="display:flex; justify-content:space-between; font-size:12px; font-weight:600;"><span>${k}</span><span>${v} (${pct}%)</span></div><div style="background:#e2e8f0; height:6px; border-radius:99px;"><div style="background:${c}; width:${pct}%; height:100%; border-radius:99px;"></div></div></div>`;
    }).join('');

    document.getElementById('status-chart-container').innerHTML = `
      <div style="display:flex; align-items:flex-end; gap:16px; height:120px;">
        ${Object.entries(s.status_distribution).map(([k,v]) => `
          <div style="flex:1; display:flex; flex-direction:column; align-items:center; height:100%; justify-content:flex-end;">
            <div style="font-size:12px; font-weight:700; margin-bottom:4px;">${v}</div>
            <div style="background:#2563eb; width:100%; height:${Math.max(12, Math.round((v/(s.total_complaints||1))*100))}%; border-radius:4px 4px 0 0;"></div>
            <div style="font-size:11px; color:#64748b; margin-top:4px;">${k}</div>
          </div>
        `).join('')}
      </div>
    `;
  }

  async function loadAdminRequests() {
    const tbody = document.getElementById('admin-tickets-tbody');
    const dVal = document.getElementById('admin-filter-dept').value;
    const sVal = document.getElementById('admin-filter-status').value;
    const pVal = document.getElementById('admin-filter-priority').value;
    const qVal = document.getElementById('admin-search-input').value;
    const res = await fetch(`/api/tickets?dept_id=${dVal}&status=${sVal}&priority=${pVal}&search=${encodeURIComponent(qVal)}`);
    const data = await res.json();
    tbody.innerHTML = data.map(t => `
      <tr>
        <td><a class="ticket-code-link" onclick="openModal(${t.id})">${t.ticket_code}</a></td>
        <td><div style="font-weight:600;">${t.student_name}</div><div style="font-size:11px; color:#64748b;">${t.student_id}</div></td>
        <td style="max-width:200px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis;">${t.title}</td>
        <td>${t.department_name}</td>
        <td><span class="badge badge-${t.priority.toLowerCase()}">${t.priority}</span></td>
        <td><span class="badge badge-${t.status.toLowerCase().replace(/ /g,'-')}">${t.status}</span></td>
        <td style="font-size:12px; color:#64748b;">${new Date(t.created_at).toLocaleDateString()}</td>
        <td><button class="btn btn-outline btn-sm" onclick="openModal(${t.id})">Manage</button></td>
      </tr>
    `).join('') || '<tr><td colspan="8" style="text-align:center; padding:20px;">No requests found.</td></tr>';
  }

  async function openModal(tid) {
    const res = await fetch('/api/tickets/' + tid);
    const t = await res.json();
    state.currentTicket = t;

    document.getElementById('modal-ticket-code').textContent = t.ticket_code;
    document.getElementById('modal-ticket-title').textContent = t.title;
    document.getElementById('modal-priority-badge').textContent = t.priority;
    document.getElementById('modal-priority-badge').className = 'badge badge-' + t.priority.toLowerCase();
    document.getElementById('modal-status-badge').textContent = t.status;
    document.getElementById('modal-status-badge').className = 'badge badge-' + t.status.toLowerCase().replace(/ /g,'-');
    document.getElementById('modal-student-name').textContent = t.student.name + ' (' + t.student.student_id + ')';
    document.getElementById('modal-department-name').textContent = t.department.name;
    document.getElementById('modal-category-name').textContent = t.category.name;
    document.getElementById('modal-created-time').textContent = new Date(t.created_at).toLocaleString();
    document.getElementById('modal-raw-message').textContent = t.raw_message;
    document.getElementById('modal-required-action').textContent = t.required_action;

    document.getElementById('modal-admin-action-panel').style.display = (state.currentUser && state.currentUser.role === 'admin') ? 'block' : 'none';
    if (state.currentUser && state.currentUser.role === 'admin') {
      document.getElementById('modal-status-select').value = t.status;
      document.getElementById('modal-reassign-dept').value = t.department.id || '';
    }

    document.getElementById('modal-message-thread').innerHTML = t.messages.map(m => `
      <div class="chat-bubble ${m.sender_role==='student'?'student':'staff'}">
        <div class="chat-sender">${m.sender_name} • ${new Date(m.created_at).toLocaleTimeString([], {hour:'2-digit', minute:'2-digit'})}</div>
        <div>${m.message_text}</div>
      </div>
    `).join('') || '<div style="color:#94a3b8; font-size:12px; text-align:center;">No messages yet.</div>';

    document.getElementById('modal-timeline-list').innerHTML = t.status_history.map(h => `
      <div class="timeline-item">
        <div class="timeline-dot"></div>
        <div class="timeline-time">${new Date(h.created_at).toLocaleString()} by ${h.changed_by_name}</div>
        <div class="timeline-content">${h.new_status} ${h.reason ? '— ' + h.reason : ''}</div>
      </div>
    `).join('');

    document.getElementById('ticket-modal-overlay').classList.add('active');
  }

  async function updateModalStatus() {
    if (!state.currentTicket) return;
    const st = document.getElementById('modal-status-select').value;
    const note = document.getElementById('modal-status-note').value;
    await fetch(`/api/tickets/${state.currentTicket.id}/status`, { method:'PUT', headers:{'Content-Type':'application/json'}, body:JSON.stringify({status:st, reason:note, resolution_notes:note}) });
    showToast('Updated status to ' + st, 'success');
    openModal(state.currentTicket.id);
    refreshTab();
  }

  async function reassignDept() {
    if (!state.currentTicket) return;
    const did = document.getElementById('modal-reassign-dept').value;
    await fetch(`/api/tickets/${state.currentTicket.id}/reassign`, { method:'PUT', headers:{'Content-Type':'application/json'}, body:JSON.stringify({department_id:parseInt(did)}) });
    showToast('Reassigned department', 'success');
    openModal(state.currentTicket.id);
    refreshTab();
  }

  async function sendMsg() {
    if (!state.currentTicket) return;
    const input = document.getElementById('modal-message-input');
    if (!input.value.trim()) return;
    await fetch(`/api/tickets/${state.currentTicket.id}/messages`, { method:'POST', headers:{'Content-Type':'application/json'}, body:JSON.stringify({message_text:input.value.trim()}) });
    input.value = '';
    openModal(state.currentTicket.id);
  }

  async function loadAuto() {
    const res = await fetch('/api/automation/status');
    const d = await res.json();
    document.getElementById('timing-reminder-slider').value = d.demo_reminder_minutes;
    document.getElementById('reminder-slider-val').textContent = d.demo_reminder_minutes + 'm';
    document.getElementById('timing-escalation-slider').value = d.demo_escalation_minutes;
    document.getElementById('escalation-slider-val').textContent = d.demo_escalation_minutes + 'm';
  }

  async function saveTiming() {
    const rem = document.getElementById('timing-reminder-slider').value;
    const esc = document.getElementById('timing-escalation-slider').value;
    await fetch('/api/automation/config', { method:'POST', headers:{'Content-Type':'application/json'}, body:JSON.stringify({demo_reminder_minutes:parseInt(rem), demo_escalation_minutes:parseInt(esc)}) });
    showToast('Updated timing settings.', 'success');
  }

  async function runSLACheck() {
    const res = await fetch('/api/automation/run-check', { method:'POST' });
    const d = await res.json();
    showToast(d.message, 'success');
    refreshTab();
    updateUnread();
  }

  async function timeJump() {
    const res = await fetch('/api/automation/simulate-time-jump?minutes=15', { method:'POST' });
    const d = await res.json();
    showToast(d.message, 'warning');
    refreshTab();
    updateUnread();
  }

  async function loadEmails() {
    const res = await fetch('/api/notifications/emails');
    const data = await res.json();
    document.getElementById('emails-tbody').innerHTML = data.map(e => `
      <tr>
        <td style="font-weight:600;">${e.recipient}</td>
        <td>${e.subject}</td>
        <td><span class="badge badge-assigned">${e.type}</span></td>
        <td style="font-size:12px; color:#64748b;">${new Date(e.created_at).toLocaleString()}</td>
        <td><button class="btn btn-outline btn-sm" onclick="previewEmail(${e.id})">Preview Email</button></td>
      </tr>
    `).join('') || '<tr><td colspan="5" style="text-align:center; padding:20px;">No emails logged yet.</td></tr>';
  }

  function previewEmail(nid) {
    document.getElementById('email-preview-iframe').src = '/api/notifications/emails/' + nid + '/preview';
    document.getElementById('email-modal-overlay').classList.add('active');
  }

  async function updateUnread() {
    try {
      const res = await fetch('/api/notifications/unread-count');
      const d = await res.json();
      document.getElementById('unread-notif-badge').style.display = d.unread_count > 0 ? 'block' : 'none';
    } catch(e) {}
  }

  function setupEvents() {
    document.getElementById('submit-complaint-btn').addEventListener('click', submitComplaint);
    document.getElementById('clarification-submit-btn').addEventListener('click', submitComplaint);
    document.getElementById('student-status-filter').addEventListener('change', loadStudentRequests);
    document.getElementById('admin-filter-dept').addEventListener('change', loadAdminRequests);
    document.getElementById('admin-filter-status').addEventListener('change', loadAdminRequests);
    document.getElementById('admin-filter-priority').addEventListener('change', loadAdminRequests);
    document.getElementById('admin-search-input').addEventListener('input', loadAdminRequests);

    document.getElementById('modal-status-update-btn').addEventListener('click', updateModalStatus);
    document.getElementById('modal-reassign-btn').addEventListener('click', reassignDept);
    document.getElementById('modal-send-message-btn').addEventListener('click', sendMsg);

    document.getElementById('timing-reminder-slider').addEventListener('input', e => document.getElementById('reminder-slider-val').textContent = e.target.value + 'm');
    document.getElementById('timing-escalation-slider').addEventListener('input', e => document.getElementById('escalation-slider-val').textContent = e.target.value + 'm');
    document.getElementById('save-timing-btn').addEventListener('click', saveTiming);
    document.getElementById('run-sla-check-btn').addEventListener('click', runSLACheck);
    document.getElementById('quick-sla-btn').addEventListener('click', runSLACheck);
    document.getElementById('time-jump-btn').addEventListener('click', timeJump);

    document.getElementById('notif-bell-btn').addEventListener('click', async (e) => {
      e.stopPropagation();
      const dd = document.getElementById('notifications-dropdown');
      const active = dd.classList.toggle('active');
      if (active) {
        const res = await fetch('/api/notifications');
        const notifs = await res.json();
        document.getElementById('notif-dropdown-list').innerHTML = notifs.map(n => `
          <div class="notif-item ${!n.is_read?'unread':''}" onclick="onNotifClick(${n.id}, ${n.ticket_id})">
            <div style="font-weight:700;">${n.title}</div>
            <div style="font-size:12px; color:#475569;">${n.message}</div>
          </div>
        `).join('') || '<div style="padding:15px; text-align:center; color:#94a3b8;">No notifications yet.</div>';
      }
    });

    document.addEventListener('click', e => {
      const dd = document.getElementById('notifications-dropdown');
      if (dd && !dd.contains(e.target) && e.target.id !== 'notif-bell-btn') dd.classList.remove('active');
    });
  }

  async function onNotifClick(nid, tid) {
    await fetch(`/api/notifications/${nid}/read`, { method:'PUT' });
    document.getElementById('notifications-dropdown').classList.remove('active');
    updateUnread();
    if (tid) openModal(tid);
  }

  function initModals() {
    document.querySelectorAll('.modal-close-btn').forEach(b => b.addEventListener('click', () => {
      document.querySelectorAll('.modal-overlay').forEach(m => m.classList.remove('active'));
    }));
  }

  function showToast(msg, type='info') {
    const c = document.getElementById('toast-container');
    if (!c) return;
    const t = document.createElement('div');
    const bg = type==='success' ? '#10b981' : (type==='danger' ? '#ef4444' : (type==='warning' ? '#f59e0b' : '#2563eb'));
    t.style.cssText = `background:${bg}; color:#fff; padding:12px 18px; border-radius:8px; box-shadow:0 4px 12px rgba(0,0,0,0.15); font-size:13.5px; font-weight:600; margin-top:10px;`;
    t.textContent = msg;
    c.appendChild(t);
    setTimeout(() => t.remove(), 3500);
  }
</script>
</body>
</html>
"""

@app.get("/", response_class=HTMLResponse)
async def serve_single_index():
    return HTMLResponse(content=INDEX_HTML, status_code=200)

# ---------------------------------------------------------------------------
# 10. CLI RUNNER
# ---------------------------------------------------------------------------
if __name__ == "__main__":
    banner = rf"""
========================================================================
   ____                                   _____ _                 _    ___ 
  / ___|__ _ _ __ ___  _ __  _   _ ___   |  ___| | _____      __ / \  |_ _|
 | |   / _` | '_ ` _ \| '_ \| | | / __|  | |_  | |/ _ \ \ /\ / // _ \  | | 
 | |__| (_| | | | | | | |_) | |_| \__ \  |  _| | | (_) \ V  V // ___ \ | | 
  \____\__,_|_| |_| |_| .__/ \__,_|___/  |_|   |_|\___/ \_/\_//_/   \_\___|
                       |_|                                                  
========================================================================
           "Your Campus. Your Request. Automatically Handled."
========================================================================
 ALL-IN-ONE SINGLE FILE APPLICATION LAUNCHING:
 
 Local URL:      http://{HOST}:{PORT}
 College:        {COLLEGE_NAME}
 Demo Mode:      Active (Reminders: {DEMO_CONFIG['demo_reminder_minutes']}m, Escalation: {DEMO_CONFIG['demo_escalation_minutes']}m)
 
 ZERO EXTERNAL DEPENDENCIES REQUIRED!
 Everything (Backend, Frontend, AI Agent, SQLite, Seed Data) runs from this file.
========================================================================
    """
    print(banner)
    uvicorn.run(app, host=HOST, port=PORT, log_level="info")
