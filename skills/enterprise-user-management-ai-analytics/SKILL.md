---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "how do I set up the enterprise user management system"
  - "integrate AI analytics for user management"
  - "create a user dashboard with task tracking"
  - "implement AI-based ticket classification"
  - "build admin dashboard with user management"
  - "add burnout detection and risk prediction"
  - "set up JWT authentication for user system"
  - "configure ML service for user analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system that combines React frontend, Node.js backend, and FastAPI ML service to provide intelligent user administration, task tracking, support ticket management, and AI-powered analytics including risk detection, anomaly detection, and burnout analysis.

## Project Architecture

The system consists of three main components:

1. **Frontend** (React.js) - User and admin dashboards
2. **Backend** (Node.js + Express) - REST API and business logic
3. **ML Service** (FastAPI + scikit-learn) - AI analytics and predictions

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install
```

Create `.env` file:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Runs at http://localhost:3000
```

## Backend API Reference

### Authentication Routes

```javascript
// Register new user
POST /api/auth/register
Body: {
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
Body: {
  "email": "john@example.com",
  "password": "securepass123"
}
Response: {
  "token": "jwt_token_here",
  "user": { "id": "...", "name": "...", "role": "..." }
}

// Get current user
GET /api/auth/me
Headers: { "Authorization": "Bearer <token>" }
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <admin_token>" }

// Create user
POST /api/users
Body: {
  "name": "Jane Smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Body: { "name": "Jane Doe", "status": "active" }

// Delete user
DELETE /api/users/:userId
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: { "Authorization": "Bearer <token>" }

// Create task (Admin)
POST /api/tasks
Body: {
  "title": "Implement feature X",
  "description": "Add new dashboard widget",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:taskId/status
Body: { "status": "in-progress" } // or "done"

// Log time on task
POST /api/tasks/:taskId/time
Body: {
  "hours": 2.5,
  "description": "Completed initial implementation"
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Body: {
  "title": "Cannot access dashboard",
  "description": "Getting 403 error",
  "priority": "medium",
  "category": "technical"
}

// Get tickets
GET /api/tickets?status=open&priority=high

// Update ticket
PATCH /api/tickets/:ticketId
Body: { "status": "in-progress", "assignedTo": "admin_id" }

// Add comment
POST /api/tickets/:ticketId/comments
Body: { "comment": "Investigating the issue..." }
```

## ML Service API

### AI-Powered Ticket Classification

```python
# FastAPI endpoint for ticket classification
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib

app = FastAPI()

class TicketRequest(BaseModel):
    title: str
    description: str

@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketRequest):
    """
    Classifies ticket into categories: technical, billing, access, general
    Returns category and confidence score
    """
    text = f"{ticket.title} {ticket.description}"
    
    # Load pre-trained model
    model = joblib.load("models/ticket_classifier.pkl")
    vectorizer = joblib.load("models/tfidf_vectorizer.pkl")
    
    # Predict
    features = vectorizer.transform([text])
    category = model.predict(features)[0]
    confidence = model.predict_proba(features).max()
    
    # Route to appropriate team
    routing = {
        "technical": "IT Support",
        "billing": "Finance Team",
        "access": "Admin Team",
        "general": "General Support"
    }
    
    return {
        "category": category,
        "confidence": float(confidence),
        "routeTo": routing[category],
        "priority": "high" if confidence > 0.8 else "medium"
    }
```

### Risk Prediction

```python
from river import linear_model, preprocessing
from datetime import datetime

class RiskPredictor:
    def __init__(self):
        self.model = linear_model.LogisticRegression()
        self.scaler = preprocessing.StandardScaler()
    
    def extract_features(self, user_data):
        """Extract risk features from user behavior"""
        return {
            "failed_logins": user_data.get("failedLoginAttempts", 0),
            "unusual_hours": user_data.get("afterHoursAccess", 0),
            "data_downloads": user_data.get("largeDownloads", 0),
            "policy_violations": user_data.get("violations", 0),
            "access_changes": user_data.get("permissionChanges", 0)
        }

@app.post("/api/ml/predict-risk")
async def predict_risk(user_id: str):
    """
    Predicts user risk score based on behavior patterns
    """
    # Fetch user activity from backend
    user_data = await fetch_user_activity(user_id)
    
    predictor = RiskPredictor()
    features = predictor.extract_features(user_data)
    
    # Calculate risk score (0-100)
    risk_score = (
        features["failed_logins"] * 15 +
        features["unusual_hours"] * 10 +
        features["data_downloads"] * 20 +
        features["policy_violations"] * 30 +
        features["access_changes"] * 15
    )
    
    risk_level = "high" if risk_score > 70 else "medium" if risk_score > 40 else "low"
    
    return {
        "userId": user_id,
        "riskScore": min(risk_score, 100),
        "riskLevel": risk_level,
        "factors": features,
        "recommendations": get_recommendations(risk_level)
    }
```

### Burnout Detection

```python
from datetime import datetime, timedelta
import numpy as np

@app.post("/api/ml/detect-burnout")
async def detect_burnout(user_id: str):
    """
    Analyzes workload patterns to detect potential burnout
    """
    # Get last 30 days of activity
    activities = await fetch_user_activities(user_id, days=30)
    
    # Calculate metrics
    total_hours = sum(a.get("hoursWorked", 0) for a in activities)
    avg_daily_hours = total_hours / 30
    overtime_days = sum(1 for a in activities if a.get("hoursWorked", 0) > 8)
    weekend_work = sum(1 for a in activities if a.get("isWeekend"))
    task_completion_rate = calculate_completion_rate(activities)
    
    # Burnout indicators
    burnout_score = 0
    indicators = []
    
    if avg_daily_hours > 9:
        burnout_score += 30
        indicators.append("Excessive daily hours")
    
    if overtime_days > 15:
        burnout_score += 25
        indicators.append("Frequent overtime")
    
    if weekend_work > 4:
        burnout_score += 20
        indicators.append("Weekend work pattern")
    
    if task_completion_rate < 0.7:
        burnout_score += 25
        indicators.append("Declining productivity")
    
    return {
        "userId": user_id,
        "burnoutScore": burnout_score,
        "riskLevel": "high" if burnout_score > 60 else "medium" if burnout_score > 30 else "low",
        "indicators": indicators,
        "metrics": {
            "avgDailyHours": round(avg_daily_hours, 2),
            "overtimeDays": overtime_days,
            "weekendWork": weekend_work,
            "completionRate": round(task_completion_rate, 2)
        },
        "suggestions": [
            "Consider workload redistribution",
            "Schedule time off",
            "Review task priorities"
        ] if burnout_score > 60 else []
    }
```

### Anomaly Detection

```python
from sklearn.ensemble import IsolationForest
import pandas as pd

@app.post("/api/ml/detect-anomalies")
async def detect_anomalies(user_id: str = None):
    """
    Detects unusual patterns in user behavior
    """
    # Fetch user activities
    if user_id:
        activities = await fetch_user_activities(user_id)
    else:
        activities = await fetch_all_activities()
    
    # Prepare features
    df = pd.DataFrame([
        {
            "login_hour": a.get("loginHour"),
            "session_duration": a.get("sessionMinutes"),
            "actions_count": a.get("actionsCount"),
            "data_accessed": a.get("dataAccessedMB"),
            "failed_attempts": a.get("failedAttempts")
        }
        for a in activities
    ])
    
    # Train isolation forest
    model = IsolationForest(contamination=0.1, random_state=42)
    predictions = model.fit_predict(df)
    
    # Get anomalies
    anomalies = []
    for idx, pred in enumerate(predictions):
        if pred == -1:  # Anomaly detected
            activity = activities[idx]
            anomalies.append({
                "userId": activity["userId"],
                "timestamp": activity["timestamp"],
                "type": classify_anomaly_type(activity),
                "severity": calculate_severity(activity),
                "details": activity
            })
    
    return {
        "totalActivities": len(activities),
        "anomaliesDetected": len(anomalies),
        "anomalies": anomalies
    }
```

## Frontend Components

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUser = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(res.data.user);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    const { token, user } = res.data;
    localStorage.setItem('token', token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    setUser(user);
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### Kanban Board Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './TaskBoard.css';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`);
      const grouped = {
        todo: res.data.filter(t => t.status === 'todo'),
        inProgress: res.data.filter(t => t.status === 'in-progress'),
        done: res.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable onDragStart={(e) => e.dataTransfer.setData('taskId', task._id)}>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-badge ${task.priority}`}>{task.priority}</span>
      <div className="task-actions">
        {task.status !== 'done' && (
          <button onClick={() => moveTask(task._id, 
            task.status === 'todo' ? 'in-progress' : 'done')}>
            Move →
          </button>
        )}
      </div>
    </div>
  );

  const Column = ({ title, tasks, status }) => (
    <div 
      className="kanban-column"
      onDrop={(e) => {
        const taskId = e.dataTransfer.getData('taskId');
        moveTask(taskId, status);
      }}
      onDragOver={(e) => e.preventDefault()}
    >
      <h3>{title} ({tasks.length})</h3>
      {tasks.map(task => <TaskCard key={task._id} task={task} />)}
    </div>
  );

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} status="todo" />
      <Column title="In Progress" tasks={tasks.inProgress} status="in-progress" />
      <Column title="Done" tasks={tasks.done} status="done" />
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { LineChart, Line, BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, Legend } from 'recharts';

const AIAnalytics = () => {
  const [riskData, setRiskData] = useState([]);
  const [burnoutData, setBurnoutData] = useState(null);
  const [anomalies, setAnomalies] = useState([]);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch risk predictions
      const riskRes = await axios.get(`${process.env.REACT_APP_ML_API_URL}/api/ml/users/risk-scores`);
      setRiskData(riskRes.data);

      // Fetch burnout analysis
      const burnoutRes = await axios.get(`${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-analysis`);
      setBurnoutData(burnoutRes.data);

      // Fetch anomalies
      const anomalyRes = await axios.get(`${process.env.REACT_APP_ML_API_URL}/api/ml/detect-anomalies`);
      setAnomalies(anomalyRes.data.anomalies);
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI-Powered Analytics</h2>

      <div className="analytics-section">
        <h3>User Risk Scores</h3>
        <BarChart width={600} height={300} data={riskData}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis dataKey="userName" />
          <YAxis />
          <Tooltip />
          <Legend />
          <Bar dataKey="riskScore" fill="#ff6b6b" />
        </BarChart>
      </div>

      <div className="analytics-section">
        <h3>Burnout Detection</h3>
        {burnoutData && (
          <div className="burnout-metrics">
            <div className="metric">
              <span>High Risk Users:</span>
              <strong>{burnoutData.highRiskCount}</strong>
            </div>
            <div className="metric">
              <span>Avg Daily Hours:</span>
              <strong>{burnoutData.avgDailyHours}</strong>
            </div>
            <div className="metric">
              <span>Overtime Frequency:</span>
              <strong>{burnoutData.overtimePercentage}%</strong>
            </div>
          </div>
        )}
      </div>

      <div className="analytics-section">
        <h3>Anomalies Detected ({anomalies.length})</h3>
        <table className="anomalies-table">
          <thead>
            <tr>
              <th>User</th>
              <th>Type</th>
              <th>Severity</th>
              <th>Timestamp</th>
            </tr>
          </thead>
          <tbody>
            {anomalies.map((anomaly, idx) => (
              <tr key={idx} className={`severity-${anomaly.severity}`}>
                <td>{anomaly.userId}</td>
                <td>{anomaly.type}</td>
                <td>{anomaly.severity}</td>
                <td>{new Date(anomaly.timestamp).toLocaleString()}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Common Patterns

### Protected Routes

```javascript
// src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  if (!user) return <Navigate to="/login" />;
  if (adminOnly && user.role !== 'admin') return <Navigate to="/dashboard" />;

  return children;
};

export default ProtectedRoute;
```

### API Error Handling

```javascript
// src/utils/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection failed:', error.message);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Configuration

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Model Not Loading

```python
# ml-service/utils/model_loader.py
import os
import joblib
from pathlib import Path

def load_model(model_name):
    model_path = Path(os.getenv('MODEL_PATH', './models')) / f"{model_name}.pkl"
    
    if not model_path.exists():
        raise FileNotFoundError(f"Model {model_name} not found at {model_path}")
    
    return joblib.load(model_path)
```

### Token Expiration Handling

```javascript
// frontend/src/utils/refreshToken.js
import axios from 'axios';

export const setupTokenRefresh = () => {
  setInterval(async () => {
    const token = localStorage.getItem('token');
    if (token) {
      try {
        const res = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/refresh`, { token });
        localStorage.setItem('token', res.data.token);
      } catch (error) {
        console.error('Token refresh failed');
      }
    }
  }, 6 * 60 * 60 * 1000); // Refresh every 6 hours
};
```

## Configuration Files

### Backend package.json Scripts

```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "seed": "node scripts/seedData.js"
  }
}
```

### ML Service requirements.txt

```
fastapi==0.104.1
uvicorn==0.24.0
scikit-learn==1.3.2
river==0.19.0
pandas==2.1.3
numpy==1.26.2
joblib==1.3.2
python-dotenv==1.0.0
```

This skill provides comprehensive coverage of the Enterprise User Management System, enabling AI agents to assist developers with implementation, integration, and troubleshooting of all system components.
