+-------------------+
|      Next.js      |
|   Frontend (UI)   |
+-------------------+
          |
          v
+-------------------+      +-----------+           +----------------+
|    FastAPI        |----->| PostgreSQL |<-------> |  pgvector/     |
|  Backend (API)    |      +-----------+           |  Weaviate      |
+-------------------+          ^  ^               +----------------+
        |   |  |               |  |
        |   |  |               |  +---AWS S3 (evidence)
        |   |  | Celery Tasks
        |   v  |
  +-------------+    (REST API, async jobs, auth, logging)
  |   Celery    |-----> NLP/LLM API (OpenAI/Anthropic, in-house)
  +-------------+
[User] 1--* [Candidate] *--* [Case] 1--* [Evidence]
   |               |             |
 1 |          * [Consent]        * [AuditLog]
   |
   * [AdminLog]
fastapi
uvicorn
sqlalchemy
psycopg2-binary
alembic
pydantic
celery
redis
boto3
jinja2
python-multipart
aiohttp
pdfkit
pgvector
openai
python-dotenv
from sqlalchemy import (Column, String, Integer, DateTime, Boolean, ForeignKey, Text, JSON)
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from pgvector.sqlalchemy import Vector

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True, index=True)
    hashed_password = Column(String)
    role = Column(String)  # admin, recruiter

class Candidate(Base):
    __tablename__ = 'candidates'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)
    phone = Column(String)
    resume_url = Column(String)
    created_at = Column(DateTime, default=func.now())
    consent_id = Column(Integer, ForeignKey('consent.id'))
    cases = relationship("Case", back_populates="candidate")

class Consent(Base):
    __tablename__ = 'consent'
    id = Column(Integer, primary_key=True)
    candidate_id = Column(Integer, ForeignKey('candidates.id'))
    signed_at = Column(DateTime, default=func.now())
    ip_address = Column(String)
    consent_text = Column(Text)
    signature = Column(String) # base64 image or hash

class Case(Base):
    __tablename__ = 'cases'
    id = Column(Integer, primary_key=True)
    candidate_id = Column(Integer, ForeignKey('candidates.id'))
    status = Column(String, default='pending')
    risk_score = Column(Integer)
    result_json = Column(JSON)
    created_at = Column(DateTime, default=func.now())
    updated_at = Column(DateTime)
    candidate = relationship("Candidate", back_populates="cases")
    evidences = relationship("Evidence", back_populates="case")

class Evidence(Base):
    __tablename__ = 'evidences'
    id = Column(Integer, primary_key=True)
    case_id = Column(Integer, ForeignKey('cases.id'))
    category = Column(String)
    snippet = Column(Text)
    source_url = Column(String)
    severity = Column(Integer)
    recency = Column(Integer)
    screenshot_url = Column(String)
    vector_repr = Column(Vector(1536))
    case = relationship("Case", back_populates="evidences")

class AuditLog(Base):
    __tablename__ = 'audit_logs'
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('users.id'))
    action = Column(String)
    meta = Column(JSON)
    created_at = Column(DateTime, default=func.now())
from fastapi import FastAPI, UploadFile, File, Form, Depends, HTTPException
from sqlalchemy.orm import Session
from database import SessionLocal, engine
from models import Base, Candidate, Consent, Case
from uuid import uuid4
import boto3, os
from celery_app import run_screening_job

app = FastAPI()

# Dependency
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.post("/api/intake")
async def intake(
    name: str = Form(...), 
    email: str = Form(...), 
    phone: str = Form(...), 
    consent: str = Form(...),
    signature: str = Form(...),
    resume: UploadFile = File(...),
    db: Session = Depends(get_db)):
    # Upload resume to S3
    s3_client = boto3.client('s3')
    filename = f"resumes/{uuid4()}_{resume.filename}"
    s3_client.upload_fileobj(resume.file, os.getenv('S3_BUCKET'), filename)
    resume_url = f"s3://{os.getenv('S3_BUCKET')}/{filename}"

    # Create candidate record
    candidate = Candidate(name=name, email=email, phone=phone, resume_url=resume_url)
    db.add(candidate)
    db.commit()
    db.refresh(candidate)

    # Store consent
    cons = Consent(candidate_id=candidate.id, consent_text=consent, signature=signature)
    db.add(cons)
    db.commit()

    # Link consent
    candidate.consent_id = cons.id
    db.commit()

    # Create screening case
    case = Case(candidate_id=candidate.id)
    db.add(case)
    db.commit()
    db.refresh(case)

    # Background screening job
    run_screening_job.delay(case.id)
    return {"case_id": case.id, "candidate_id": candidate.id}

@app.get("/api/case/{case_id}")
def get_case_result(case_id: int, db: Session = Depends(get_db)):
    case = db.query(Case).filter(Case.id == case_id).first()
    if not case:
        raise HTTPException(404, "Case not found")
    return {
        "status": case.status,
        "risk_score": case.risk_score,
        "result": case.result_json,
        "evidences": [
            {
                "category": e.category,
                "snippet": e.snippet,
                "source_url": e.source_url,
                "severity": e.severity,
                "recency": e.recency,
                "screenshot_url": e.screenshot_url
            } 
            for e in case.evidences
        ]
    }
import { useEffect, useState } from 'react';
import { useRouter } from 'next/router';

export default function Case() {
  const { query } = useRouter();
  const [data, setData] = useState(null);

  useEffect(() => {
    if (!query.id) return;
    fetch(`/api/case/${query.id}`).then(res => res.json()).then(setData);
  }, [query.id]);

  if (!data) return <div>Loading...</div>;
  return (
    <div>
      <h2>Screening Result</h2>
      <p>Status: {data.status}</p>
      <p>Risk Score: {data.risk_score}</p>
      <ul>
        {data.evidences.map((e, i) => (
          <li key={i}>{e.category} — {e.snippet} — <a href={e.source_url}>Source</a></li>
        ))}
      </ul>
    </div>
  );
}
// Auth logic omitted for brevity
export default function AdminDashboard() {
  // Fetch and render candidates/cases with links to PDFs/HTML reports, audit logs, etc.
  // Use fetch('/api/cases') and render as table
  return (
    <div className="p-8">
      <h1 className="text-2xl">Admin Dashboard</h1>
      {/* Table of cases/candidates */}
    </div>
  );
}
from jinja2 import Environment, FileSystemLoader
import pdfkit
import datetime

def generate_report(case, evidences, output_pdf='report.pdf'):
    env = Environment(loader=FileSystemLoader('templates'))
    template = env.get_template('report.html')
    html_content = template.render(
        case=case, evidences=evidences, ts=datetime.datetime.now()
    )
    pdfkit.from_string(html_content, output_pdf)
    return output_pdf

# In templates/report.html, style the report (include compliance disclaimers, timestamp, evidence snippets, etc)
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_intake_api():
    response = client.post("/api/intake", data={
        "name": "John Doe", "email": "john@example.com", "phone": "1234567",
        "consent": "I consent", "signature": "John D"
    }, files={"resume": ("resume.pdf", b"resume content", "application/pdf")})
    assert response.status_code == 200
    assert "case_id" in response.json()
