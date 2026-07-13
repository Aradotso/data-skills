---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement JWT authentication with role-based access"
  - "create AI-powered ticket classification system"
  - "build kanban board with task tracking"
  - "add burnout detection and risk prediction"
  - "configure MongoDB for user management app"
  - "deploy user management system with FastAPI ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript/Python application that combines user administration, task management, and support ticketing with AI-powered insights. The system provides risk detection, anomaly detection, burnout analysis, and predictive project insights using machine learning models.

**Architecture:**
- **Frontend:** React.js with JWT authentication
- **Backend:** Node.js with REST APIs
- **ML Service:** FastAPI with scikit-learn and River for online learning
- **Database:** MongoDB

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB installed and running
mongod --version
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt

# Install frontend dependencies
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env):**
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/user_management
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
```

**Frontend (.env):**
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

**ML Service (.env):**
```bash
# ml-service/.env
MONGODB_URI=mongodb://localhost:27017/user_management
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Running the Application

### Start All Services

```bash
# Terminal 1: Backend
cd backend
npm start

# Terminal 2: ML Service
cd ml-service
uvicorn main:app --reload --port 8000

# Terminal 3: Frontend
cd frontend
npm start
```

The application will be available at:
- Frontend: http://localhost:3000
- Backend API: http://localhost:5000
- ML Service: http://localhost:8000

## Backend API Reference

### Authentication Endpoints

```javascript
// Register new user
POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "SecurePass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "SecurePass123"
}
// Returns: { token: "jwt_token", user: {...} }

// Verify token
GET /api/auth/verify
Headers: { "Authorization": "Bearer <token>" }
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <admin_token>" }

// Create user
POST /api/users
{
  "username": "jane.smith",
  "email": "jane@example.com",
  "password": "Pass123",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
{
  "username": "jane.smith.updated",
  "role": "admin"
}

// Delete user
DELETE /api/users/:userId
```

### Task Management

```javascript
// Create task
POST /api/tasks
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Get user tasks
GET /api/tasks/user/:userId

// Update task status
PUT /api/tasks/:taskId/status
{
  "status": "in-progress" // or "done"
}

// Track time on task
POST /api/tasks/:taskId/time
{
  "duration": 3600, // seconds
  "date": "2026-04-15"
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "subject": "Cannot access dashboard",
  "description": "Getting 404 error",
  "priority": "high",
  "createdBy": "user_id"
}

// Get tickets (filtered)
GET /api/tickets?status=open&priority=high

// Update ticket
PUT /api/tickets/:ticketId
{
  "status": "resolved",
  "assignedTo": "admin_id",
  "resolution": "Fixed routing issue"
}
```

## ML Service API Reference

### AI-Powered Ticket Classification

```python
# POST http://localhost:8000/api/ml/classify-ticket
import requests

response = requests.post(
    "http://localhost:8000/api/ml/classify-ticket",
    json={
        "subject": "Cannot login to system",
        "description": "Getting authentication error repeatedly",
        "metadata": {
            "user_department": "Engineering",
            "reported_time": "2026-04-15T10:30:00Z"
        }
    }
)

# Response:
# {
#   "category": "authentication",
#   "priority": "high",
#   "suggested_assignee": "admin_id",
#   "confidence": 0.89
# }
```

### Risk Prediction

```python
# POST http://localhost:8000/api/ml/predict-risk
response = requests.post(
    "http://localhost:8000/api/ml/predict-risk",
    json={
        "user_id": "user_123",
        "behavior_metrics": {
            "login_frequency": 2.5,  # per day
            "failed_login_attempts": 8,
            "unusual_access_times": 5,
            "data_access_volume": 1500  # MB
        }
    }
)

# Response:
# {
#   "risk_score": 0.78,
#   "risk_level": "high",
#   "factors": ["high_failed_logins", "unusual_access_pattern"],
#   "recommendations": ["Reset password", "Review access logs"]
# }
```

### Burnout Detection

```python
# POST http://localhost:8000/api/ml/detect-burnout
response = requests.post(
    "http://localhost:8000/api/ml/detect-burnout",
    json={
        "user_id": "user_123",
        "workload_data": {
            "tasks_assigned": 25,
            "tasks_completed": 15,
            "avg_daily_hours": 11.5,
            "missed_deadlines": 4,
            "overtime_hours": 45  # per month
        }
    }
)

# Response:
# {
#   "burnout_risk": "high",
#   "burnout_score": 0.82,
#   "indicators": ["excessive_hours", "high_missed_deadlines"],
#   "suggestions": ["Redistribute workload", "Schedule time off"]
# }
```

### Anomaly Detection

```python
# POST http://localhost:8000/api/ml/detect-anomaly
response = requests.post(
    "http://localhost:8000/api/ml/detect-anomaly",
    json={
        "user_id": "user_123",
        "activity_log": [
            {"timestamp": "2026-04-15T02:30:00Z", "action": "login", "ip": "192.168.1.50"},
            {"timestamp": "2026-04-15T02:35:00Z", "action": "access_sensitive_data", "resource": "/api/admin/users"},
            {"timestamp": "2026-04-15T02:40:00Z", "action": "bulk_download", "size_mb": 500}
        ]
    }
)

# Response:
# {
#   "is_anomaly": true,
#   "anomaly_score": 0.91,
#   "anomalies_detected": ["unusual_time", "unauthorized_access", "large_download"],
#   "alert_level": "critical"
# }
```

### Project Delay Prediction

```python
# POST http://localhost:8000/api/ml/predict-delay
response = requests.post(
    "http://localhost:8000/api/ml/predict-delay",
    json={
        "project_id": "proj_456",
        "metrics": {
            "total_tasks": 50,
            "completed_tasks": 20,
            "days_elapsed": 30,
            "total_duration": 90,
            "team_size": 5,
            "avg_velocity": 0.67  # tasks/day
        }
    }
)

# Response:
# {
#   "delay_predicted": true,
#   "estimated_delay_days": 15,
#   "completion_probability": 0.65,
#   "risk_factors": ["low_velocity", "high_task_backlog"]
# }
```

## Frontend Integration Patterns

### React Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';
import axios from 'axios';

const AuthContext = createContext(null);

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      verifyToken(token);
    } else {
      setLoading(false);
    }
  }, []);

  const verifyToken = async (token) => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/auth/verify`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setUser(response.data.user);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(
      `${process.env.REACT_APP_API_URL}/api/auth/login`,
      { email, password }
    );
    localStorage.setItem('token', response.data.token);
    setUser(response.data.user);
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
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

### Task Management Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      const grouped = response.data.reduce((acc, task) => {
        const status = task.status === 'in-progress' ? 'inProgress' : task.status;
        acc[status] = [...(acc[status] || []), task];
        return acc;
      }, { todo: [], inProgress: [], done: [] });
      
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {Object.entries(tasks).map(([status, taskList]) => (
        <div key={status} className="kanban-column">
          <h3>{status.replace(/([A-Z])/g, ' $1').trim()}</h3>
          {taskList.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => updateTaskStatus(task._id, 'in-progress')}>
                Move to In Progress
              </button>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard Component

```javascript
// frontend/src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAIAnalytics();
  }, [userId]);

  const fetchAIAnalytics = async () => {
    try {
      // Get risk prediction
      const riskResponse = await axios.post(
        `${process.env.REACT_APP_ML_URL}/api/ml/predict-risk`,
        {
          user_id: userId,
          behavior_metrics: await getUserBehaviorMetrics(userId)
        }
      );

      // Get burnout detection
      const burnoutResponse = await axios.post(
        `${process.env.REACT_APP_ML_URL}/api/ml/detect-burnout`,
        {
          user_id: userId,
          workload_data: await getUserWorkloadData(userId)
        }
      );

      setAnalytics({
        riskScore: riskResponse.data,
        burnoutRisk: burnoutResponse.data,
        anomalies: []
      });
    } catch (error) {
      console.error('Failed to fetch AI analytics:', error);
    }
  };

  return (
    <div className="ai-analytics-dashboard">
      <div className="analytics-card">
        <h3>Risk Assessment</h3>
        {analytics.riskScore && (
          <div>
            <p>Risk Level: {analytics.riskScore.risk_level}</p>
            <p>Score: {(analytics.riskScore.risk_score * 100).toFixed(1)}%</p>
            <ul>
              {analytics.riskScore.factors.map((factor, idx) => (
                <li key={idx}>{factor}</li>
              ))}
            </ul>
          </div>
        )}
      </div>

      <div className="analytics-card">
        <h3>Burnout Detection</h3>
        {analytics.burnoutRisk && (
          <div>
            <p>Risk: {analytics.burnoutRisk.burnout_risk}</p>
            <p>Score: {(analytics.burnoutRisk.burnout_score * 100).toFixed(1)}%</p>
            <ul>
              {analytics.burnoutRisk.suggestions.map((suggestion, idx) => (
                <li key={idx}>{suggestion}</li>
              ))}
            </ul>
          </div>
        )}
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Backend Implementation Patterns

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return res.status(401).json({ message: 'No token provided' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### User Model (MongoDB/Mongoose)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.taskId,
      { status: req.body.status },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Dict, Optional
import joblib
import numpy as np
from datetime import datetime

app = FastAPI(title="User Management ML Service")

# Load pre-trained models
ticket_classifier = None  # Load your model here
risk_predictor = None
burnout_detector = None

class TicketData(BaseModel):
    subject: str
    description: str
    metadata: Optional[Dict] = {}

class BehaviorMetrics(BaseModel):
    login_frequency: float
    failed_login_attempts: int
    unusual_access_times: int
    data_access_volume: float

class RiskRequest(BaseModel):
    user_id: str
    behavior_metrics: BehaviorMetrics

@app.post("/api/ml/classify-ticket")
async def classify_ticket(data: TicketData):
    try:
        # Extract features from text
        text = f"{data.subject} {data.description}"
        
        # Simple keyword-based classification (replace with actual ML model)
        categories = {
            "authentication": ["login", "password", "access", "auth"],
            "technical": ["error", "bug", "crash", "broken"],
            "account": ["profile", "settings", "update", "change"],
            "billing": ["payment", "invoice", "subscription", "charge"]
        }
        
        text_lower = text.lower()
        detected_category = "general"
        max_matches = 0
        
        for category, keywords in categories.items():
            matches = sum(1 for keyword in keywords if keyword in text_lower)
            if matches > max_matches:
                max_matches = matches
                detected_category = category
        
        priority = "high" if any(word in text_lower for word in ["urgent", "critical", "down", "error"]) else "medium"
        
        return {
            "category": detected_category,
            "priority": priority,
            "confidence": min(0.95, 0.5 + (max_matches * 0.15)),
            "suggested_assignee": None
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskRequest):
    try:
        metrics = request.behavior_metrics
        
        # Calculate risk score (simple weighted formula - replace with ML model)
        risk_score = (
            (metrics.failed_login_attempts * 0.3) +
            (metrics.unusual_access_times * 0.2) +
            (metrics.data_access_volume / 1000 * 0.3) +
            ((10 - metrics.login_frequency) * 0.2)
        ) / 10
        
        risk_score = min(1.0, max(0.0, risk_score))
        
        risk_level = "low" if risk_score < 0.3 else "medium" if risk_score < 0.7 else "high"
        
        factors = []
        if metrics.failed_login_attempts > 5:
            factors.append("high_failed_logins")
        if metrics.unusual_access_times > 3:
            factors.append("unusual_access_pattern")
        if metrics.data_access_volume > 1000:
            factors.append("high_data_access")
        
        return {
            "risk_score": risk_score,
            "risk_level": risk_level,
            "factors": factors,
            "recommendations": ["Review access logs"] if risk_level == "high" else []
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class WorkloadData(BaseModel):
    tasks_assigned: int
    tasks_completed: int
    avg_daily_hours: float
    missed_deadlines: int
    overtime_hours: float

class BurnoutRequest(BaseModel):
    user_id: str
    workload_data: WorkloadData

@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutRequest):
    try:
        data = request.workload_data
        
        # Calculate burnout score
        burnout_score = (
            (data.avg_daily_hours / 12 * 0.3) +
            (data.missed_deadlines / 10 * 0.2) +
            (data.overtime_hours / 50 * 0.3) +
            ((data.tasks_assigned - data.tasks_completed) / data.tasks_assigned * 0.2)
        )
        
        burnout_score = min(1.0, max(0.0, burnout_score))
        
        burnout_risk = "low" if burnout_score < 0.4 else "medium" if burnout_score < 0.7 else "high"
        
        indicators = []
        if data.avg_daily_hours > 10:
            indicators.append("excessive_hours")
        if data.missed_deadlines > 2:
            indicators.append("high_missed_deadlines")
        if data.overtime_hours > 40:
            indicators.append("high_overtime")
        
        suggestions = []
        if burnout_risk in ["medium", "high"]:
            suggestions.append("Redistribute workload")
            suggestions.append("Schedule time off")
        
        return {
            "burnout_risk": burnout_risk,
            "burnout_score": burnout_score,
            "indicators": indicators,
            "suggestions": suggestions
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}
```

## Database Schema

### MongoDB Collections

```javascript
// Users Collection
{
  "_id": ObjectId,
  "username": "john.doe",
  "email": "john@example.com",
  "password": "hashed_password",
  "role": "user",
  "department": "Engineering",
  "createdAt": ISODate,
  "lastLogin": ISODate
}

// Tasks Collection
{
  "_id": ObjectId,
  "title": "Implement feature X",
  "description": "Detailed description",
  "assignedTo": ObjectId, // User reference
  "createdBy": ObjectId,
  "status": "in-progress",
  "priority": "high",
  "dueDate": ISODate,
  "timeTracked": 7200, // seconds
  "createdAt": ISODate,
  "updatedAt": ISODate
}

// Tickets Collection
{
  "_id": ObjectId,
  "subject": "Login issue",
  "description": "Cannot login",
  "priority": "high",
  "status": "open",
  "category": "authentication",
  "createdBy": ObjectId,
  "assignedTo": ObjectId,
  "resolution": null,
  "createdAt": ISODate,
  "resolvedAt": null
}

// AuditLogs Collection
{
  "_id": ObjectId,
  "userId": ObjectId,
  "action": "login",
  "resource": "/api/users",
  "ip": "192.168.1.100",
  "timestamp": ISODate,
  "metadata": {}
}
```

## Common Workflows

### Complete User Registration and Task Assignment

```javascript
// 1. Register new user
const registerUser = async (userData) => {
  const response = await axios.post(
    `${process.env.REACT_APP_API_URL}/api/auth/register`,
    userData
  );
  return response.data;
};

// 2. Admin assigns task
const assignTask = async (taskData, adminToken) => {
  const response = await axios.post(
    `${process.env.REACT_APP_API_URL}/api/tasks`,
    taskData,
    { headers: { Authorization: `Bearer ${adminToken}` } }
  );
  return response.data;
};

// 3. User tracks time on task
const trackTime = async (taskId, duration, userToken) => {
  const response = await axios.post(
    `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`,
    { duration, date: new Date().toISOString() },
    { headers: { Authorization: `Bearer ${userToken}` } }
  );
  return response.data;
};

// 4. Check for burnout risk
const checkBurnout = async (userId) => {
  const workloadData = await getUserWorkload(userId);
  const response = await axios.post(
    `${process.env.REACT_APP_ML_URL}/api/ml/detect-burnout`,
    { user_id: userId, workload_data: workloadData }
  );
  return response.data;
};
```

### AI-Powered Ticket Routing

```javascript
// Complete ticket workflow with AI classification
const handleNewTicket = async (ticketData) => {
  // 1. Create ticket
  const ticket = await axios.post(
    `${process.env.REACT_APP_API_URL}/api/tickets`,
    ticketData,
    { headers: { Authorization: `Bearer ${token}` } }
  );

  // 2. Get AI classification
  const classification = await axios.post(
    `${process.env.REACT_APP_ML_URL}/api/ml/classify-ticket`,
    {
      subject: ticketData.subject,
      description: ticketData.description
    }
  );

  // 3. Update ticket with AI suggestions
  await axios.put(
    `${process.env.REACT_APP_API_URL}/api/tickets/${ticket.data._id}`,
    {
      category: classification.data.category,
      priority: classification.data.priority,
      assignedTo: classification.data.suggested_assignee
    },
    { headers: { Authorization: `Bearer ${token}` } }
  );

  return { ticket: ticket.data, classification: classification.data };
};
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check if MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Check connection string in .env
MONGODB_URI=mongodb://localhost:27017/user_management

# Test connection with mongo shell
mongosh "mongodb://localhost:27017/user_management"
```

### JWT Token Errors

```javascript
// Clear expired tokens
localStorage.removeItem('token');

// Verify JWT_SECRET is set in backend .env
console.log(process.env.JWT_SECRET ? 'JWT_SECRET is set' : 'JWT_SECRET missing');

// Check token expiration
const jwt = require('jsonwebtoken');
const decoded = jwt.decode(token);
console.log('Token expires:', new Date(decoded.exp * 1000));
```

### ML Service Not Responding

```bash
# Check if FastAPI is running
curl http://localhost:8000/health

# Check Python dependencies
pip list | grep -E "fastapi|scikit-learn|river"

# View ML service logs
cd ml-service
uvicorn main:app --reload --log-level debug

# Test ML endpoint directly
curl -X POST http://localhost:8000/api/ml/classify-ticket \
  -H "Content-Type: application/json" \
  -d '{"subject":"test","description":"test ticket"}'
```
