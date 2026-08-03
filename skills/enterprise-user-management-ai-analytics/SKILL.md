---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and workforce insights
triggers:
  - "Set up enterprise user management system with AI analytics"
  - "How do I integrate AI analytics for user management?"
  - "Create a task management system with burnout detection"
  - "Build user dashboard with Kanban board and time tracking"
  - "Implement AI ticket classification and risk prediction"
  - "Deploy enterprise management system with MongoDB and FastAPI"
  - "Configure JWT authentication for user management app"
  - "Add anomaly detection to employee monitoring system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack JavaScript/Python application that combines user management, task tracking, and support ticket systems with AI-powered analytics. It provides intelligent insights including risk detection, anomaly detection, burnout analysis, and predictive project management through machine learning models.

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

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

Create `.env` file in `backend/` directory:

```env
PORT=5000
MONGODB_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
NODE_ENV=development
```

Start backend server:

```bash
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in `ml-service/` directory:

```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in `frontend/` directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Runs at http://localhost:3000
```

## Architecture

### Component Structure

- **Frontend (React)**: User interface, dashboards, Kanban boards
- **Backend (Node.js/Express)**: REST API, authentication, business logic
- **ML Service (FastAPI)**: AI analytics, predictions, classification
- **Database (MongoDB)**: User data, tasks, tickets, logs

## Key API Endpoints

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": { "id": "...", "email": "...", "role": "..." }
}
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Create user
POST /api/users
Headers: { "Authorization": "Bearer <token>" }
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:id
Headers: { "Authorization": "Bearer <token>" }
{
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
Headers: { "Authorization": "Bearer <token>" }
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: { "Authorization": "Bearer <token>" }

// Create task
POST /api/tasks
Headers: { "Authorization": "Bearer <token>" }
{
  "title": "Implement authentication",
  "description": "Add JWT-based authentication",
  "assignedTo": "userId123",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PUT /api/tasks/:id
Headers: { "Authorization": "Bearer <token>" }
{
  "status": "in-progress",
  "timeSpent": 120 // minutes
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { "Authorization": "Bearer <token>" }
{
  "subject": "Login issue",
  "description": "Cannot access dashboard after password reset",
  "priority": "high",
  "category": "technical"
}

// Get tickets (admin sees all, users see own)
GET /api/tickets
Headers: { "Authorization": "Bearer <token>" }

// Update ticket
PUT /api/tickets/:id
Headers: { "Authorization": "Bearer <token>" }
{
  "status": "in-progress",
  "assignedTo": "adminId",
  "response": "Investigating the issue"
}
```

### AI Analytics Endpoints

```python
# Classify ticket priority and category
POST /api/ml/classify-ticket
{
  "subject": "System crash during data export",
  "description": "Application crashes when exporting large datasets"
}

# Response
{
  "priority": "high",
  "category": "technical",
  "confidence": 0.89,
  "suggestedAssignee": "techSupportTeam"
}

# Predict user risk score
POST /api/ml/predict-risk
{
  "userId": "user123",
  "loginAttempts": 5,
  "failedLogins": 3,
  "lastActive": "2026-04-15T10:30:00Z",
  "activityPattern": [0.2, 0.5, 0.8, 0.3]
}

# Response
{
  "riskScore": 0.72,
  "riskLevel": "medium",
  "factors": ["multiple failed logins", "unusual activity pattern"],
  "recommendation": "monitor closely"
}

# Detect burnout risk
POST /api/ml/burnout-analysis
{
  "userId": "user123",
  "tasksCompleted": 45,
  "averageWorkHours": 52,
  "weekendWork": 8,
  "taskOverdueRate": 0.15
}

# Response
{
  "burnoutScore": 0.68,
  "riskLevel": "moderate",
  "indicators": ["excessive work hours", "weekend work"],
  "recommendations": ["redistribute tasks", "schedule time off"]
}

# Project delay prediction
POST /api/ml/predict-delay
{
  "projectId": "proj123",
  "tasksTotal": 50,
  "tasksCompleted": 20,
  "daysElapsed": 30,
  "daysRemaining": 20,
  "teamSize": 5
}

# Response
{
  "delayProbability": 0.75,
  "estimatedDelay": 10, // days
  "bottlenecks": ["insufficient resources", "high complexity tasks"],
  "suggestions": ["add team member", "reduce scope"]
}
```

## Frontend Integration Examples

### Setting up API Client with Authentication

```javascript
// src/utils/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to requests
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Handle auth errors
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### User Dashboard Component

```javascript
// src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import api from '../utils/api';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [tickets, setTickets] = useState([]);
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const [tasksRes, ticketsRes, analyticsRes] = await Promise.all([
        api.get('/api/tasks'),
        api.get('/api/tickets'),
        api.get('/api/analytics/user')
      ]);

      // Group tasks by status
      const grouped = tasksRes.data.reduce((acc, task) => {
        const status = task.status.replace('-', '');
        acc[status] = acc[status] || [];
        acc[status].push(task);
        return acc;
      }, { todo: [], inProgress: [], done: [] });

      setTasks(grouped);
      setTickets(ticketsRes.data);
      setAnalytics(analyticsRes.data);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await api.put(`/api/tasks/${taskId}`, { status: newStatus });
      fetchDashboardData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {/* Analytics Overview */}
      {analytics && (
        <div className="analytics-cards">
          <div className="card">
            <h3>Tasks Completed</h3>
            <p>{analytics.tasksCompleted}</p>
          </div>
          <div className="card">
            <h3>Avg Completion Time</h3>
            <p>{analytics.avgCompletionTime}h</p>
          </div>
          {analytics.burnoutRisk > 0.6 && (
            <div className="card warning">
              <h3>Burnout Risk</h3>
              <p>{(analytics.burnoutRisk * 100).toFixed(0)}%</p>
            </div>
          )}
        </div>
      )}

      {/* Kanban Board */}
      <div className="kanban-board">
        {Object.entries(tasks).map(([status, taskList]) => (
          <div key={status} className="kanban-column">
            <h2>{status.replace(/([A-Z])/g, ' $1').trim()}</h2>
            {taskList.map(task => (
              <div key={task._id} className="task-card">
                <h3>{task.title}</h3>
                <p>{task.description}</p>
                <div className="task-meta">
                  <span className={`priority ${task.priority}`}>
                    {task.priority}
                  </span>
                  <span>Due: {new Date(task.dueDate).toLocaleDateString()}</span>
                </div>
                <select 
                  value={task.status}
                  onChange={(e) => updateTaskStatus(task._id, e.target.value)}
                >
                  <option value="todo">To Do</option>
                  <option value="in-progress">In Progress</option>
                  <option value="done">Done</option>
                </select>
              </div>
            ))}
          </div>
        ))}
      </div>

      {/* Support Tickets */}
      <div className="tickets-section">
        <h2>My Tickets</h2>
        {tickets.map(ticket => (
          <div key={ticket._id} className="ticket-item">
            <h3>{ticket.subject}</h3>
            <span className={`status ${ticket.status}`}>{ticket.status}</span>
            <p>{ticket.description}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Panel with AI Insights

```javascript
// src/components/AdminPanel.jsx
import React, { useState, useEffect } from 'react';
import api from '../utils/api';
import axios from 'axios';

const AdminPanel = () => {
  const [users, setUsers] = useState([]);
  const [aiInsights, setAiInsights] = useState([]);
  const [alerts, setAlerts] = useState([]);

  const mlApi = axios.create({
    baseURL: process.env.REACT_APP_ML_API_URL
  });

  useEffect(() => {
    fetchAdminData();
  }, []);

  const fetchAdminData = async () => {
    try {
      const [usersRes, alertsRes] = await Promise.all([
        api.get('/api/users'),
        api.get('/api/admin/alerts')
      ]);

      setUsers(usersRes.data);
      setAlerts(alertsRes.data);

      // Fetch AI insights for at-risk users
      await fetchAIInsights(usersRes.data);
    } catch (error) {
      console.error('Error fetching admin data:', error);
    }
  };

  const fetchAIInsights = async (userList) => {
    const insights = await Promise.all(
      userList.map(async (user) => {
        try {
          const riskRes = await mlApi.post('/api/ml/predict-risk', {
            userId: user._id,
            loginAttempts: user.loginAttempts || 0,
            failedLogins: user.failedLogins || 0,
            lastActive: user.lastActive
          });

          const burnoutRes = await mlApi.post('/api/ml/burnout-analysis', {
            userId: user._id,
            tasksCompleted: user.tasksCompleted || 0,
            averageWorkHours: user.avgWorkHours || 40,
            weekendWork: user.weekendWork || 0,
            taskOverdueRate: user.overdueRate || 0
          });

          return {
            userId: user._id,
            userName: user.name,
            risk: riskRes.data,
            burnout: burnoutRes.data
          };
        } catch (error) {
          console.error(`Error fetching insights for ${user.name}:`, error);
          return null;
        }
      })
    );

    setAiInsights(insights.filter(Boolean));
  };

  const assignTicketWithAI = async (ticketId, ticketData) => {
    try {
      // Get AI classification
      const classifyRes = await mlApi.post('/api/ml/classify-ticket', {
        subject: ticketData.subject,
        description: ticketData.description
      });

      // Update ticket with AI suggestions
      await api.put(`/api/tickets/${ticketId}`, {
        priority: classifyRes.data.priority,
        category: classifyRes.data.category,
        assignedTo: classifyRes.data.suggestedAssignee
      });

      console.log('Ticket auto-assigned:', classifyRes.data);
    } catch (error) {
      console.error('Error in AI ticket assignment:', error);
    }
  };

  const deleteUser = async (userId) => {
    if (!window.confirm('Are you sure you want to delete this user?')) return;
    
    try {
      await api.delete(`/api/users/${userId}`);
      fetchAdminData();
    } catch (error) {
      console.error('Error deleting user:', error);
    }
  };

  return (
    <div className="admin-panel">
      <h1>Admin Dashboard</h1>

      {/* AI Alerts */}
      {aiInsights.filter(i => i.risk.riskLevel === 'high' || i.burnout.riskLevel === 'high').length > 0 && (
        <div className="ai-alerts">
          <h2>⚠️ AI Alerts</h2>
          {aiInsights.filter(i => i.risk.riskLevel === 'high' || i.burnout.riskLevel === 'high').map(insight => (
            <div key={insight.userId} className="alert-card">
              <h3>{insight.userName}</h3>
              {insight.risk.riskLevel === 'high' && (
                <p className="risk">Security Risk: {(insight.risk.riskScore * 100).toFixed(0)}%</p>
              )}
              {insight.burnout.riskLevel === 'high' && (
                <p className="burnout">Burnout Risk: {(insight.burnout.burnoutScore * 100).toFixed(0)}%</p>
              )}
              <p className="recommendations">
                {[...insight.risk.recommendations, ...insight.burnout.recommendations].join(', ')}
              </p>
            </div>
          ))}
        </div>
      )}

      {/* User Management */}
      <div className="users-table">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Risk Score</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => {
              const insight = aiInsights.find(i => i.userId === user._id);
              return (
                <tr key={user._id}>
                  <td>{user.name}</td>
                  <td>{user.email}</td>
                  <td>{user.role}</td>
                  <td>{user.status}</td>
                  <td className={insight?.risk.riskLevel}>
                    {insight ? (insight.risk.riskScore * 100).toFixed(0) + '%' : 'N/A'}
                  </td>
                  <td>
                    <button onClick={() => deleteUser(user._id)}>Delete</button>
                  </td>
                </tr>
              );
            })}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AdminPanel;
```

### Time Tracking Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect, useRef } from 'react';
import api from '../utils/api';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setElapsedTime(prev => prev + 1);
      }, 1000);
    } else if (intervalRef.current) {
      clearInterval(intervalRef.current);
    }

    return () => {
      if (intervalRef.current) clearInterval(intervalRef.current);
    };
  }, [isRunning]);

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const saveTimeLog = async () => {
    try {
      await api.post('/api/time-logs', {
        taskId,
        duration: elapsedTime,
        date: new Date()
      });

      // Update task with time spent
      await api.put(`/api/tasks/${taskId}`, {
        timeSpent: elapsedTime / 60 // Convert to minutes
      });

      setElapsedTime(0);
      setIsRunning(false);
    } catch (error) {
      console.error('Error saving time log:', error);
    }
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(elapsedTime)}</div>
      <div className="timer-controls">
        <button onClick={() => setIsRunning(!isRunning)}>
          {isRunning ? 'Pause' : 'Start'}
        </button>
        <button onClick={() => setElapsedTime(0)}>Reset</button>
        <button onClick={saveTimeLog} disabled={elapsedTime === 0}>
          Save
        </button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

## ML Service Implementation

### FastAPI ML Service Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models (create these directories first)
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str

class TicketClassificationResponse(BaseModel):
    priority: str
    category: str
    confidence: float
    suggestedAssignee: Optional[str] = None

class RiskPredictionRequest(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    lastActive: str
    activityPattern: Optional[List[float]] = None

class RiskPredictionResponse(BaseModel):
    riskScore: float
    riskLevel: str
    factors: List[str]
    recommendation: str

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    tasksCompleted: int
    averageWorkHours: float
    weekendWork: float
    taskOverdueRate: float

class BurnoutAnalysisResponse(BaseModel):
    burnoutScore: float
    riskLevel: str
    indicators: List[str]
    recommendations: List[str]

@app.post("/api/ml/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    """Classify ticket priority and category using NLP"""
    try:
        # Simple rule-based classification (replace with trained model)
        text = (request.subject + " " + request.description).lower()
        
        # Priority classification
        high_priority_keywords = ['urgent', 'critical', 'crash', 'down', 'emergency', 'immediately']
        medium_priority_keywords = ['issue', 'problem', 'bug', 'error']
        
        if any(keyword in text for keyword in high_priority_keywords):
            priority = "high"
            confidence = 0.85
        elif any(keyword in text for keyword in medium_priority_keywords):
            priority = "medium"
            confidence = 0.75
        else:
            priority = "low"
            confidence = 0.65
        
        # Category classification
        if any(word in text for word in ['login', 'password', 'access', 'authentication']):
            category = "authentication"
        elif any(word in text for word in ['crash', 'error', 'bug', 'technical']):
            category = "technical"
        elif any(word in text for word in ['account', 'billing', 'payment']):
            category = "billing"
        else:
            category = "general"
        
        # Suggest assignee based on category
        assignee_map = {
            "authentication": "securityTeam",
            "technical": "techSupportTeam",
            "billing": "financeTeam",
            "general": "supportTeam"
        }
        
        return TicketClassificationResponse(
            priority=priority,
            category=category,
            confidence=confidence,
            suggestedAssignee=assignee_map.get(category, "supportTeam")
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    """Predict security risk based on user behavior"""
    try:
        risk_score = 0.0
        factors = []
        
        # Failed login analysis
        if request.failedLogins > 3:
            risk_score += 0.3
            factors.append("multiple failed logins")
        
        # Login attempt frequency
        if request.loginAttempts > 10:
            risk_score += 0.2
            factors.append("high login frequency")
        
        # Last active time
        try:
            last_active = datetime.fromisoformat(request.lastActive.replace('Z', '+00:00'))
            hours_inactive = (datetime.now(last_active.tzinfo) - last_active).total_seconds() / 3600
            
            if hours_inactive > 168:  # 1 week
                risk_score += 0.15
                factors.append("prolonged inactivity")
        except:
            pass
        
        # Activity pattern anomaly
        if request.activityPattern:
            pattern_variance = np.var(request.activityPattern)
            if pattern_variance > 0.5:
                risk_score += 0.25
                factors.append("unusual activity pattern")
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = "high"
            recommendation = "immediate investigation required"
        elif risk_score >= 0.4:
            risk_level = "medium"
            recommendation = "monitor closely"
        else:
            risk_level = "low"
            recommendation = "standard monitoring"
        
        return RiskPredictionResponse(
            riskScore=min(risk_score, 1.0),
            riskLevel=risk_level,
            factors=factors if factors else ["normal activity"],
            recommendation=recommendation
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis", response_model=BurnoutAnalysisResponse)
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze employee burnout risk"""
    try:
        burnout_score = 0.0
        indicators = []
        recommendations = []
        
        # Work hours analysis
        if request.averageWorkHours > 50:
            burnout_score += 0.3
            indicators.append("excessive work hours")
            recommendations.append("reduce work hours to sustainable levels")
        
        # Weekend work
        if request.weekendWork > 4:
            burnout_score += 0.25
            indicators.append("weekend work")
            recommendations.append("limit weekend work")
        
        # Task overdue rate
        if request.taskOverdueRate > 0.2:
            burnout_score += 0.2
            indicators.append("high task overdue rate")
            recommendations.append("redistribute tasks")
        
        # Task completion intensity
        if request.tasksCompleted > 40:
            burnout_score += 0.15
            indicators.append("high task volume")
            recommendations.append("balance workload")
        
        # Determine risk level
        if burnout_score >= 0.6:
            risk_level = "high"
            recommendations.append("schedule time off")
            recommendations.append("consider workload redistribution")
        elif burnout_score >= 0.3:
            risk_level = "moderate"
            recommendations.append("monitor work-life balance")
        else:
            risk_level = "low"
            recommendations.append("maintain current pace")
        
        return BurnoutAnalysisResponse(
            burnoutScore=min(burnout_score, 1.0),
            riskLevel=risk_level,
            indicators=indicators if indicators else ["healthy work pattern"],
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

### ML Service Requirements

```txt
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
numpy==1.24.3
scikit-learn==1.3.2
river==0.19.0
joblib==1.3.2
python-dotenv==1.0.0
pandas==2.1.3
```

## Backend API Implementation

### User Model and Routes

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please provide a name'],
    trim: true
  },
  email: {
    type: String,
    required: [true, 'Please provide an email'],
    unique: true,
    lowercase: true,
    match: [/^\S+@\S+\.\S+$/, 'Please provide a valid email']
  },
  password: {
    type: String,
    required: [true, 'Please provide a password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  loginAttempts: { type: Number, default: 0 },
  failedLogins: { type: Number, default: 0 },
  lastActive: { type: Date, default: Date.now },
  tasksCompleted: { type: Number, default: 0 },
  avgWorkHours: { type: Number, default: 40 },
  weekendWork: { type: Number, default: 0 },
  overdueRate: { type: Number, default: 0 }
}, {
  timestamps: true
});

// Hash password before saving
