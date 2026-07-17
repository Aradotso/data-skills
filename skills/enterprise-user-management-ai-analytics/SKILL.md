---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing using React, Node.js, and FastAPI ML services
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics into user management
  - implement JWT authentication for user roles
  - create AI-powered ticket classification
  - build a kanban task board with user tracking
  - set up burnout detection and risk prediction
  - configure MongoDB for user management system
  - deploy enterprise user management with ML service
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack application that combines traditional user/task management with AI-powered insights. It provides role-based access control (Admin/User), task tracking with Kanban boards, support ticket management, and ML-driven analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights.

**Core Components:**
- **Frontend**: React.js application for user/admin dashboards
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based AI analytics engine using scikit-learn and River
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

### Clone and Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env  # Configure environment variables

# ML Service setup
cd ../ml-service
pip install -r requirements.txt
cp .env.example .env  # Configure ML service

# Frontend setup
cd ../frontend
npm install
cp .env.example .env  # Configure API endpoints
```

### Environment Configuration

**Backend (.env)**:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**:
```env
PORT=8000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

**Frontend (.env)**:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the Application

```bash
# Terminal 1 - Backend
cd backend
npm start  # Runs on http://localhost:5000

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload  # Runs on http://localhost:8000

# Terminal 3 - Frontend
cd frontend
npm start  # Runs on http://localhost:3000
```

## Backend API Reference

### Authentication

**User Registration**:
```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'  // 'admin' or 'user'
    })
  });
  return response.json();
};
```

**User Login**:
```javascript
// POST /api/auth/login
const loginUser = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

**Get All Users**:
```javascript
// GET /api/users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

**Update User**:
```javascript
// PUT /api/users/:id
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

**Delete User**:
```javascript
// DELETE /api/users/:id
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management

**Create Task**:
```javascript
// POST /api/tasks
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority,  // 'low', 'medium', 'high'
      status: 'todo',  // 'todo', 'in-progress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};
```

**Update Task Status**:
```javascript
// PATCH /api/tasks/:id/status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};
```

**Get User Tasks**:
```javascript
// GET /api/tasks/user/:userId
const getUserTasks = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Support Tickets

**Create Ticket**:
```javascript
// POST /api/tickets
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category  // AI will classify if not provided
    })
  });
  return response.json();
};
```

**Get All Tickets (Admin)**:
```javascript
// GET /api/tickets
const getAllTickets = async (filters = {}) => {
  const token = localStorage.getItem('token');
  const queryParams = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tickets?${queryParams}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## ML Service API Reference

### Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      title: "Issue with login"
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', confidence: 0.87, priority: 'high' }
  return result;
};
```

### Risk Prediction

```javascript
// POST /api/ml/predict-risk
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      login_frequency: 45,
      failed_logins: 3,
      task_completion_rate: 0.65,
      avg_task_time: 120
    })
  });
  const result = await response.json();
  // Returns: { risk_score: 0.72, risk_level: 'medium', factors: [...] }
  return result;
};
```

### Burnout Detection

```javascript
// POST /api/ml/detect-burnout
const detectBurnout = async (userMetrics) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userMetrics.userId,
      avg_daily_hours: userMetrics.avgDailyHours,
      task_overload_ratio: userMetrics.taskOverloadRatio,
      missed_deadlines: userMetrics.missedDeadlines,
      weekend_work_frequency: userMetrics.weekendWork
    })
  });
  const result = await response.json();
  // Returns: { burnout_risk: 'high', score: 0.81, recommendations: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/detect-anomaly
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: activityData.userId,
      login_time: activityData.loginTime,
      ip_address: activityData.ipAddress,
      location: activityData.location,
      actions_per_minute: activityData.actionsPerMinute
    })
  });
  const result = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.92, type: 'suspicious_location' }
  return result;
};
```

### Project Delay Prediction

```javascript
// POST /api/ml/predict-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.projectId,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      days_remaining: projectData.daysRemaining,
      team_size: projectData.teamSize,
      avg_completion_rate: projectData.avgCompletionRate
    })
  });
  const result = await response.json();
  // Returns: { delay_probability: 0.68, estimated_delay_days: 5, factors: [...] }
  return result;
};
```

## Frontend Patterns

### React Component for Kanban Board

```javascript
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchUserTasks();
  }, [userId]);

  const fetchUserTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    // Group tasks by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchUserTasks();
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} onMove={moveTask} />
      <Column title="In Progress" tasks={tasks.inProgress} onMove={moveTask} />
      <Column title="Done" tasks={tasks.done} onMove={moveTask} />
    </div>
  );
};

export default KanbanBoard;
```

### AI-Powered Ticket Creation

```javascript
import React, { useState } from 'react';

const CreateTicketWithAI = () => {
  const [ticketData, setTicketData] = useState({
    title: '',
    description: ''
  });
  const [aiSuggestions, setAiSuggestions] = useState(null);

  const handleAIClassify = async () => {
    // Get AI classification
    const mlResponse = await fetch('http://localhost:8000/api/ml/classify-ticket', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        text: ticketData.description,
        title: ticketData.title
      })
    });
    const suggestions = await mlResponse.json();
    setAiSuggestions(suggestions);
  };

  const submitTicket = async () => {
    const token = localStorage.getItem('token');
    await fetch('http://localhost:5000/api/tickets', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        ...ticketData,
        category: aiSuggestions?.category,
        priority: aiSuggestions?.priority
      })
    });
  };

  return (
    <div className="ticket-form">
      <input
        value={ticketData.title}
        onChange={(e) => setTicketData({ ...ticketData, title: e.target.value })}
        placeholder="Ticket Title"
      />
      <textarea
        value={ticketData.description}
        onChange={(e) => setTicketData({ ...ticketData, description: e.target.value })}
        placeholder="Description"
      />
      <button onClick={handleAIClassify}>Get AI Suggestions</button>
      {aiSuggestions && (
        <div className="ai-suggestions">
          <p>Category: {aiSuggestions.category}</p>
          <p>Priority: {aiSuggestions.priority}</p>
          <p>Confidence: {(aiSuggestions.confidence * 100).toFixed(0)}%</p>
        </div>
      )}
      <button onClick={submitTicket}>Submit Ticket</button>
    </div>
  );
};

export default CreateTicketWithAI;
```

### Admin Dashboard with AI Analytics

```javascript
import React, { useEffect, useState } from 'react';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    users: [],
    riskUsers: [],
    burnoutAlerts: [],
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    
    // Get all users
    const usersRes = await fetch('http://localhost:5000/api/users', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const users = await usersRes.json();

    // Get AI analytics for each user
    const aiPromises = users.map(async (user) => {
      const riskRes = await fetch('http://localhost:8000/api/ml/predict-risk', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: user._id, ...user.metrics })
      });
      return { user, risk: await riskRes.json() };
    });

    const aiResults = await Promise.all(aiPromises);
    const highRisk = aiResults.filter(r => r.risk.risk_level === 'high');

    setAnalytics({
      users,
      riskUsers: highRisk,
      burnoutAlerts: [],  // Fetch burnout data similarly
      anomalies: []  // Fetch anomaly data similarly
    });
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      <div className="stats">
        <div>Total Users: {analytics.users.length}</div>
        <div>High Risk Users: {analytics.riskUsers.length}</div>
        <div>Burnout Alerts: {analytics.burnoutAlerts.length}</div>
      </div>
      <div className="risk-users">
        <h2>High Risk Users</h2>
        {analytics.riskUsers.map(({ user, risk }) => (
          <div key={user._id} className="risk-card">
            <h3>{user.name}</h3>
            <p>Risk Score: {(risk.risk_score * 100).toFixed(0)}%</p>
            <p>Factors: {risk.factors.join(', ')}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## MongoDB Schema Examples

### User Schema

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  metrics: {
    loginFrequency: { type: Number, default: 0 },
    failedLogins: { type: Number, default: 0 },
    taskCompletionRate: { type: Number, default: 0 },
    avgTaskTime: { type: Number, default: 0 },
    avgDailyHours: { type: Number, default: 0 }
  },
  createdAt: { type: Date, default: Date.now },
  lastLogin: { type: Date }
});

module.exports = mongoose.model('User', userSchema);
```

### Task Schema

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['todo', 'in-progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  dueDate: { type: Date },
  timeSpent: { type: Number, default: 0 },  // in minutes
  createdAt: { type: Date, default: Date.now },
  completedAt: { type: Date }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Schema

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'urgent'], 
    default: 'medium' 
  },
  category: { 
    type: String, 
    enum: ['technical', 'billing', 'general', 'feature-request'] 
  },
  aiClassification: {
    category: String,
    confidence: Number,
    suggestedPriority: String
  },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## JWT Middleware Implementation

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

## Python ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Optional

app = FastAPI(title="Enterprise User Management ML Service")

# Enable CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models (train these separately)
try:
    ticket_classifier = joblib.load('./models/ticket_classifier.pkl')
    risk_predictor = joblib.load('./models/risk_predictor.pkl')
except:
    ticket_classifier = None
    risk_predictor = None

class TicketData(BaseModel):
    text: str
    title: str

class RiskData(BaseModel):
    user_id: str
    login_frequency: int
    failed_logins: int
    task_completion_rate: float
    avg_task_time: float

class BurnoutData(BaseModel):
    user_id: str
    avg_daily_hours: float
    task_overload_ratio: float
    missed_deadlines: int
    weekend_work_frequency: float

@app.post("/api/ml/classify-ticket")
async def classify_ticket(data: TicketData):
    """Classify ticket category and priority using NLP"""
    if not ticket_classifier:
        # Fallback to rule-based classification
        text_lower = (data.text + " " + data.title).lower()
        
        if any(word in text_lower for word in ['login', 'password', 'access', 'error']):
            category = 'technical'
            priority = 'high'
        elif any(word in text_lower for word in ['payment', 'invoice', 'billing']):
            category = 'billing'
            priority = 'medium'
        elif any(word in text_lower for word in ['feature', 'enhancement', 'suggestion']):
            category = 'feature-request'
            priority = 'low'
        else:
            category = 'general'
            priority = 'medium'
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.75
        }
    
    # Use trained model
    prediction = ticket_classifier.predict([data.text + " " + data.title])
    confidence = ticket_classifier.predict_proba([data.text + " " + data.title]).max()
    
    return {
        "category": prediction[0],
        "priority": "high" if confidence > 0.8 else "medium",
        "confidence": float(confidence)
    }

@app.post("/api/ml/predict-risk")
async def predict_risk(data: RiskData):
    """Predict user risk based on behavior metrics"""
    # Calculate risk score
    features = np.array([[
        data.login_frequency,
        data.failed_logins,
        data.task_completion_rate,
        data.avg_task_time
    ]])
    
    # Simple scoring algorithm
    risk_score = 0.0
    factors = []
    
    if data.failed_logins > 5:
        risk_score += 0.3
        factors.append("high_failed_logins")
    
    if data.task_completion_rate < 0.5:
        risk_score += 0.3
        factors.append("low_completion_rate")
    
    if data.login_frequency < 10:
        risk_score += 0.2
        factors.append("low_activity")
    
    if data.avg_task_time > 180:
        risk_score += 0.2
        factors.append("slow_task_completion")
    
    risk_level = "high" if risk_score > 0.6 else "medium" if risk_score > 0.3 else "low"
    
    return {
        "risk_score": risk_score,
        "risk_level": risk_level,
        "factors": factors,
        "user_id": data.user_id
    }

@app.post("/api/ml/detect-burnout")
async def detect_burnout(data: BurnoutData):
    """Detect employee burnout risk"""
    burnout_score = 0.0
    recommendations = []
    
    if data.avg_daily_hours > 9:
        burnout_score += 0.3
        recommendations.append("Reduce daily working hours")
    
    if data.task_overload_ratio > 1.2:
        burnout_score += 0.3
        recommendations.append("Reassign some tasks")
    
    if data.missed_deadlines > 3:
        burnout_score += 0.2
        recommendations.append("Review deadline feasibility")
    
    if data.weekend_work_frequency > 0.5:
        burnout_score += 0.2
        recommendations.append("Encourage work-life balance")
    
    risk_level = "high" if burnout_score > 0.6 else "medium" if burnout_score > 0.3 else "low"
    
    return {
        "burnout_risk": risk_level,
        "score": burnout_score,
        "recommendations": recommendations,
        "user_id": data.user_id
    }

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(data: dict):
    """Detect anomalous user behavior"""
    is_anomaly = False
    anomaly_score = 0.0
    anomaly_type = None
    
    # Check for suspicious patterns
    if data.get('actions_per_minute', 0) > 50:
        is_anomaly = True
        anomaly_score = 0.9
        anomaly_type = 'bot_like_activity'
    
    # Add more anomaly detection logic
    
    return {
        "is_anomaly": is_anomaly,
        "anomaly_score": anomaly_score,
        "type": anomaly_type,
        "user_id": data.get('user_id')
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Common Troubleshooting

### Issue: JWT Token Expired

```javascript
// Implement token refresh logic
const refreshToken = async () => {
  try {
    const refreshToken = localStorage.getItem('refreshToken');
    const response = await fetch('http://localhost:5000/api/auth/refresh', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken })
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    return data.token;
  } catch (error) {
    // Redirect to login
    window.location.href = '/login';
  }
};
```

### Issue: MongoDB Connection Failed

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Issue: ML Service Not Responding

Check if the ML service is running and accessible:

```bash
# Test ML service health
curl http://localhost:8000/health

# Check logs
cd ml-service
uvicorn main:app --reload --log-level debug
```

### Issue: CORS Errors

```javascript
