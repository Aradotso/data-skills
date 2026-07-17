---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "configure user task tracking with AI insights"
  - "implement role-based access control with analytics"
  - "add AI-powered ticket classification system"
  - "build user management dashboard with risk detection"
  - "create burnout detection for employee monitoring"
  - "deploy enterprise management with ML predictions"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with machine learning capabilities. It provides:

- **User Management**: JWT-based authentication, role-based access control (Admin/User)
- **Task Management**: Kanban board, time tracking, task assignment
- **Support System**: Ticket classification and intelligent routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Real-time Insights**: Performance dashboards, audit logs, security alerts

**Stack**: React.js (frontend), Node.js (backend), FastAPI (ML service), MongoDB (database)

## Installation

### Prerequisites

```bash
# Node.js and npm
node --version  # v14+ required
npm --version   # v6+ required

# Python for ML service
python --version  # 3.8+ required
pip --version

# MongoDB
mongod --version  # 4.4+ required
```

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
MONGO_URI=mongodb://localhost:27017/enterprise_user_mgmt
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Architecture

```
frontend:3000 (React)
    ↓
backend:5000 (Node.js/Express)
    ↓
ml-service:8000 (FastAPI)
    ↓
MongoDB (Database)
```

## Backend API Reference

### Authentication Endpoints

```javascript
// Register new user
POST /api/auth/register
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"  // "user" or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}

// Response
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "_id": "60f7b3b3b3b3b3b3b3b3b3b3",
    "name": "John Doe",
    "email": "john@example.com",
    "role": "user"
  }
}
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Headers: Authorization: Bearer ${JWT_TOKEN}

// Create user
POST /api/users
Headers: Authorization: Bearer ${JWT_TOKEN}
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "password": "password123",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Headers: Authorization: Bearer ${JWT_TOKEN}
{
  "name": "Jane Smith Updated",
  "department": "Product"
}

// Delete user
DELETE /api/users/:userId
Headers: Authorization: Bearer ${JWT_TOKEN}
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: Authorization: Bearer ${JWT_TOKEN}

// Create task
POST /api/tasks
Headers: Authorization: Bearer ${JWT_TOKEN}
{
  "title": "Implement authentication",
  "description": "Add JWT-based authentication to the API",
  "assignedTo": "60f7b3b3b3b3b3b3b3b3b3b3",
  "priority": "high",  // "low", "medium", "high"
  "dueDate": "2026-05-01T00:00:00.000Z",
  "status": "todo"  // "todo", "in_progress", "done"
}

// Update task status
PUT /api/tasks/:taskId
Headers: Authorization: Bearer ${JWT_TOKEN}
{
  "status": "in_progress"
}

// Track time on task
POST /api/tasks/:taskId/time
Headers: Authorization: Bearer ${JWT_TOKEN}
{
  "duration": 3600  // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: Authorization: Bearer ${JWT_TOKEN}
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when trying to access admin dashboard",
  "priority": "high",
  "category": "technical"  // "technical", "account", "feature_request"
}

// Get tickets (admin sees all, user sees own)
GET /api/tickets
Headers: Authorization: Bearer ${JWT_TOKEN}

// Update ticket
PUT /api/tickets/:ticketId
Headers: Authorization: Bearer ${JWT_TOKEN}
{
  "status": "in_progress",
  "assignedTo": "60f7b3b3b3b3b3b3b3b3b3b3"
}
```

## ML Service API Reference

### AI-Powered Ticket Classification

```python
# Python client example
import requests

ML_SERVICE_URL = "http://localhost:8000"

def classify_ticket(title, description):
    response = requests.post(
        f"{ML_SERVICE_URL}/api/ml/classify-ticket",
        json={
            "title": title,
            "description": description
        }
    )
    return response.json()

# Usage
result = classify_ticket(
    "Cannot login to system",
    "I'm getting an error message when trying to login with my credentials"
)
print(result)
# {
#   "category": "technical",
#   "priority": "high",
#   "confidence": 0.87,
#   "suggested_assignee": "support_team_1"
# }
```

### Risk Detection

```python
def detect_user_risk(user_id, user_data):
    """
    Analyze user behavior for risk indicators
    """
    response = requests.post(
        f"{ML_SERVICE_URL}/api/ml/risk-detection",
        json={
            "user_id": user_id,
            "login_attempts": user_data["login_attempts"],
            "failed_logins": user_data["failed_logins"],
            "unusual_hours": user_data["unusual_hours"],
            "data_access_volume": user_data["data_access_volume"],
            "permission_changes": user_data["permission_changes"]
        }
    )
    return response.json()

# Usage
risk_analysis = detect_user_risk("60f7b3b3b3b3b3b3b3b3b3b3", {
    "login_attempts": 15,
    "failed_logins": 8,
    "unusual_hours": True,
    "data_access_volume": 500,
    "permission_changes": 3
})
print(risk_analysis)
# {
#   "risk_score": 0.78,
#   "risk_level": "high",
#   "factors": ["multiple_failed_logins", "unusual_hours"],
#   "recommendation": "Review user access patterns and consider temporary restriction"
# }
```

### Burnout Detection

```python
def analyze_burnout(user_id, workload_data):
    """
    Detect employee burnout risk based on workload patterns
    """
    response = requests.post(
        f"{ML_SERVICE_URL}/api/ml/burnout-detection",
        json={
            "user_id": user_id,
            "tasks_completed": workload_data["tasks_completed"],
            "tasks_overdue": workload_data["tasks_overdue"],
            "avg_work_hours": workload_data["avg_work_hours"],
            "weekend_work": workload_data["weekend_work"],
            "task_completion_rate": workload_data["task_completion_rate"]
        }
    )
    return response.json()

# Usage
burnout_risk = analyze_burnout("60f7b3b3b3b3b3b3b3b3b3b3", {
    "tasks_completed": 45,
    "tasks_overdue": 12,
    "avg_work_hours": 11.5,
    "weekend_work": 8,
    "task_completion_rate": 0.65
})
print(burnout_risk)
# {
#   "burnout_score": 0.82,
#   "risk_level": "high",
#   "indicators": ["excessive_hours", "weekend_work", "declining_completion_rate"],
#   "suggestion": "Reassign tasks and schedule a wellness check-in"
# }
```

### Project Delay Prediction

```python
def predict_project_delay(project_data):
    """
    Predict likelihood of project delays
    """
    response = requests.post(
        f"{ML_SERVICE_URL}/api/ml/project-prediction",
        json={
            "project_id": project_data["project_id"],
            "total_tasks": project_data["total_tasks"],
            "completed_tasks": project_data["completed_tasks"],
            "team_size": project_data["team_size"],
            "avg_task_duration": project_data["avg_task_duration"],
            "days_remaining": project_data["days_remaining"]
        }
    )
    return response.json()

# Usage
prediction = predict_project_delay({
    "project_id": "proj_001",
    "total_tasks": 50,
    "completed_tasks": 20,
    "team_size": 5,
    "avg_task_duration": 3.5,
    "days_remaining": 30
})
print(prediction)
# {
#   "delay_probability": 0.73,
#   "estimated_delay_days": 12,
#   "completion_confidence": 0.27,
#   "recommendations": ["Add 2 team members", "Reduce scope by 15%"]
# }
```

### Anomaly Detection

```python
def detect_anomalies(activity_data):
    """
    Detect unusual patterns in user activity
    """
    response = requests.post(
        f"{ML_SERVICE_URL}/api/ml/anomaly-detection",
        json={
            "user_id": activity_data["user_id"],
            "activity_log": activity_data["activity_log"]
        }
    )
    return response.json()

# Usage
anomalies = detect_anomalies({
    "user_id": "60f7b3b3b3b3b3b3b3b3b3b3",
    "activity_log": [
        {"timestamp": "2026-04-15T02:30:00Z", "action": "data_export", "records": 5000},
        {"timestamp": "2026-04-15T02:35:00Z", "action": "permission_change", "target": "admin"},
        {"timestamp": "2026-04-15T02:40:00Z", "action": "bulk_delete", "records": 200}
    ]
})
print(anomalies)
# {
#   "anomalies_detected": True,
#   "anomaly_score": 0.91,
#   "suspicious_activities": ["unusual_time", "bulk_operations", "permission_escalation"],
#   "alert_level": "critical"
# }
```

## Frontend Integration

### React Hook for API Calls

```javascript
// hooks/useApi.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useApi = () => {
  const [token, setToken] = useState(localStorage.getItem('token'));

  const api = axios.create({
    baseURL: API_URL,
    headers: {
      'Content-Type': 'application/json',
      ...(token && { Authorization: `Bearer ${token}` })
    }
  });

  return { api, token, setToken };
};

// Usage in component
import { useApi } from '../hooks/useApi';

function TaskList() {
  const [tasks, setTasks] = useState([]);
  const { api } = useApi();

  useEffect(() => {
    const fetchTasks = async () => {
      try {
        const response = await api.get('/api/tasks');
        setTasks(response.data.tasks);
      } catch (error) {
        console.error('Error fetching tasks:', error);
      }
    };
    fetchTasks();
  }, []);

  return (
    <div>
      {tasks.map(task => (
        <div key={task._id}>{task.title}</div>
      ))}
    </div>
  );
}
```

### Admin Dashboard Component

```javascript
// components/AdminDashboard.js
import React, { useState, useEffect } from 'react';
import { useApi } from '../hooks/useApi';

function AdminDashboard() {
  const [analytics, setAnalytics] = useState(null);
  const [risks, setRisks] = useState([]);
  const { api } = useApi();

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      // Get analytics
      const analyticsRes = await api.get('/api/analytics/overview');
      setAnalytics(analyticsRes.data);

      // Get risk alerts
      const risksRes = await api.get('/api/analytics/risks');
      setRisks(risksRes.data.risks);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      {analytics && (
        <div className="analytics-grid">
          <div className="stat-card">
            <h3>Total Users</h3>
            <p>{analytics.totalUsers}</p>
          </div>
          <div className="stat-card">
            <h3>Active Tasks</h3>
            <p>{analytics.activeTasks}</p>
          </div>
          <div className="stat-card">
            <h3>Pending Tickets</h3>
            <p>{analytics.pendingTickets}</p>
          </div>
        </div>
      )}

      <div className="risk-alerts">
        <h2>Risk Alerts</h2>
        {risks.map(risk => (
          <div key={risk.userId} className={`alert alert-${risk.level}`}>
            <strong>{risk.userName}</strong>: {risk.message}
            <button onClick={() => handleRisk(risk)}>Review</button>
          </div>
        ))}
      </div>
    </div>
  );
}

export default AdminDashboard;
```

### Kanban Board Component

```javascript
// components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import { useApi } from '../hooks/useApi';

function KanbanBoard() {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });
  const { api } = useApi();

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await api.get('/api/tasks');
      const categorized = {
        todo: response.data.tasks.filter(t => t.status === 'todo'),
        in_progress: response.data.tasks.filter(t => t.status === 'in_progress'),
        done: response.data.tasks.filter(t => t.status === 'done')
      };
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await api.put(`/api/tasks/${taskId}`, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {['todo', 'in_progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h2>{status.replace('_', ' ').toUpperCase()}</h2>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h3>{task.title}</h3>
              <p>{task.description}</p>
              <div className="task-actions">
                {status !== 'done' && (
                  <button onClick={() => moveTask(
                    task._id,
                    status === 'todo' ? 'in_progress' : 'done'
                  )}>
                    Move →
                  </button>
                )}
              </div>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
}

export default KanbanBoard;
```

## Common Patterns

### Protected Routes

```javascript
// components/ProtectedRoute.js
import { Navigate } from 'react-router-dom';
import { useApi } from '../hooks/useApi';

function ProtectedRoute({ children, adminOnly = false }) {
  const { token } = useApi();
  const user = JSON.parse(localStorage.getItem('user'));

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
}

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/dashboard" element={
          <ProtectedRoute>
            <Dashboard />
          </ProtectedRoute>
        } />
        <Route path="/admin" element={
          <ProtectedRoute adminOnly>
            <AdminDashboard />
          </ProtectedRoute>
        } />
      </Routes>
    </BrowserRouter>
  );
}
```

### Real-time Notifications

```javascript
// hooks/useNotifications.js
import { useState, useEffect } from 'react';
import { useApi } from './useApi';

export const useNotifications = () => {
  const [notifications, setNotifications] = useState([]);
  const { api } = useApi();

  useEffect(() => {
    const fetchNotifications = async () => {
      try {
        const response = await api.get('/api/notifications');
        setNotifications(response.data.notifications);
      } catch (error) {
        console.error('Error fetching notifications:', error);
      }
    };

    fetchNotifications();
    const interval = setInterval(fetchNotifications, 30000); // Poll every 30s

    return () => clearInterval(interval);
  }, []);

  const markAsRead = async (notificationId) => {
    try {
      await api.put(`/api/notifications/${notificationId}/read`);
      setNotifications(prev =>
        prev.map(n => n._id === notificationId ? { ...n, read: true } : n)
      );
    } catch (error) {
      console.error('Error marking notification as read:', error);
    }
  };

  return { notifications, markAsRead };
};
```

## Backend Implementation Examples

### Express Server Setup

```javascript
// server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const authRoutes = require('./routes/auth');
const userRoutes = require('./routes/users');
const taskRoutes = require('./routes/tasks');
const ticketRoutes = require('./routes/tickets');
const analyticsRoutes = require('./routes/analytics');

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Database connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => console.log('MongoDB connected'))
  .catch(err => console.error('MongoDB connection error:', err));

// Routes
app.use('/api/auth', authRoutes);
app.use('/api/users', userRoutes);
app.use('/api/tasks', taskRoutes);
app.use('/api/tickets', ticketRoutes);
app.use('/api/analytics', analyticsRoutes);

// Error handling middleware
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ success: false, message: err.message });
});

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ success: false, message: 'Not authorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ success: false, message: 'Invalid token' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        message: `Role ${req.user.role} is not authorized`
      });
    }
    next();
  };
};
```

### Task Controller

```javascript
// controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.getTasks = async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user._id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });

    res.json({ success: true, tasks });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user._id
    });

    // Get AI prediction for task complexity
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/api/ml/predict-task-duration`,
      {
        title: task.title,
        description: task.description,
        priority: task.priority
      }
    );

    task.estimatedDuration = mlResponse.data.estimated_hours;
    await task.save();

    res.status(201).json({ success: true, task });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({ success: false, message: 'Task not found' });
    }

    // Check authorization
    if (req.user.role !== 'admin' && 
        task.assignedTo.toString() !== req.user._id.toString()) {
      return res.status(403).json({ success: false, message: 'Not authorized' });
    }

    Object.assign(task, req.body);
    await task.save();

    res.json({ success: true, task });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};
```

## ML Service Implementation

### FastAPI Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from datetime import datetime

app = FastAPI(title="Enterprise ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models at startup
try:
    ticket_classifier = joblib.load("models/ticket_classifier.pkl")
    risk_detector = joblib.load("models/risk_detector.pkl")
    burnout_model = joblib.load("models/burnout_model.pkl")
except FileNotFoundError:
    print("Warning: Model files not found. Using placeholder models.")

class TicketInput(BaseModel):
    title: str
    description: str

class RiskInput(BaseModel):
    user_id: str
    login_attempts: int
    failed_logins: int
    unusual_hours: bool
    data_access_volume: int
    permission_changes: int

class BurnoutInput(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_overdue: int
    avg_work_hours: float
    weekend_work: int
    task_completion_rate: float

@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketInput):
    """
    Classify ticket category and priority using NLP
    """
    try:
        # Feature extraction
        text_features = extract_text_features(ticket.title, ticket.description)
        
        # Predict category and priority
        category = ticket_classifier.predict_category(text_features)
        priority = ticket_classifier.predict_priority(text_features)
        confidence = ticket_classifier.predict_confidence(text_features)
        
        return {
            "category": category,
            "priority": priority,
            "confidence": float(confidence),
            "suggested_assignee": get_best_assignee(category)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/risk-detection")
async def detect_risk(data: RiskInput):
    """
    Analyze user behavior for security risks
    """
    try:
        # Prepare features
        features = np.array([
            data.login_attempts,
            data.failed_logins,
            int(data.unusual_hours),
            data.data_access_volume,
            data.permission_changes
        ]).reshape(1, -1)
        
        # Predict risk
        risk_score = risk_detector.predict_proba(features)[0][1]
        
        # Determine risk level
        if risk_score > 0.7:
            risk_level = "high"
        elif risk_score > 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        # Identify factors
        factors = []
        if data.failed_logins > 5:
            factors.append("multiple_failed_logins")
        if data.unusual_hours:
            factors.append("unusual_hours")
        if data.permission_changes > 2:
            factors.append("excessive_permission_changes")
        
        return {
            "risk_score": float(risk_score),
            "risk_level": risk_level,
            "factors": factors,
            "recommendation": get_risk_recommendation(risk_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(data: BurnoutInput):
    """
    Detect employee burnout risk
    """
    try:
        # Prepare features
        features = np.array([
            data.tasks_completed,
            data.tasks_overdue,
            data.avg_work_hours,
            data.weekend_work,
            data.task_completion_rate
        ]).reshape(1, -1)
        
        # Predict burnout
        burnout_score = burnout_model.predict_proba(features)[0][1]
        
        # Determine risk level
        if burnout_score > 0.7:
            risk_level = "high"
        elif burnout_score > 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        # Identify indicators
        indicators = []
        if data.avg_work_hours > 10:
            indicators.append("excessive_hours")
        if data.weekend_work > 4:
            indicators.append("weekend_work")
        if data.task_completion_rate < 0.7:
            indicators.append("declining_completion_rate")
        
        return {
            "burnout_score": float(burnout_score),
            "risk_level": risk_level,
            "indicators": indicators,
            "suggestion": get_burnout_suggestion(risk_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def extract_text_features(title: str, description: str):
    """Extract features from text"""
    # Implement TF-IDF or other text vectorization
    pass

def get_best_assignee(category: str):
    """Route ticket to appropriate team"""
    routing = {
        "technical": "support_team_1",
        "account": "support_team_2",
        "feature_request": "product_team"
    }
    return routing.get(category, "general_support")

def
