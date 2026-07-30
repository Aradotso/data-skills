---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, risk detection, and burnout prediction
triggers:
  - "set up enterprise user management with AI analytics"
  - "configure user management system with task tracking"
  - "integrate AI-based ticket classification and routing"
  - "implement burnout detection and risk prediction"
  - "create admin dashboard for user and task management"
  - "add AI assistant for enterprise management system"
  - "deploy user management app with kanban board"
  - "configure JWT authentication for user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack application for managing enterprise users, tasks, and support tickets with integrated AI capabilities including risk detection, anomaly detection, burnout analysis, and predictive project insights.

## What This Project Does

This system provides:
- **User Management**: Role-based access control, user CRUD operations, authentication via JWT
- **Task Management**: Kanban board (To Do/In Progress/Done), time tracking, task assignment
- **Support Tickets**: Raise, track, and auto-route tickets using AI classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, user monitoring
- **AI Assistant**: Query system for quick insights

## Installation

### Prerequisites

```bash
# Node.js 14+ and Python 3.8+ required
node --version
python --version
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
```

Create `.env` file in backend directory:

```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/enterprise_management
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Runs on http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# Runs on http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Runs on http://localhost:3000
```

## Key API Endpoints

### Authentication

**Register User**
```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/register`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role // 'user' or 'admin'
    })
  });
  return await response.json();
};
```

**Login**
```javascript
// POST /api/auth/login
const loginUser = async (credentials) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/login`, {
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

**Get All Users**
```javascript
// GET /api/users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/users`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

**Update User**
```javascript
// PUT /api/users/:id
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};
```

**Delete User**
```javascript
// DELETE /api/users/:id
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

### Task Management

**Create Task**
```javascript
// POST /api/tasks
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'inprogress', 'done'
    })
  });
  return await response.json();
};
```

**Update Task Status**
```javascript
// PATCH /api/tasks/:id/status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status: newStatus })
  });
  return await response.json();
};
```

**Track Time**
```javascript
// POST /api/tasks/:id/time
const trackTime = async (taskId, timeData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      startTime: timeData.startTime,
      endTime: timeData.endTime,
      duration: timeData.duration // in minutes
    })
  });
  return await response.json();
};
```

### Support Tickets

**Create Ticket**
```javascript
// POST /api/tickets
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tickets`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return await response.json();
};
```

**AI Ticket Classification**
```javascript
// POST /api/tickets/classify
const classifyTicket = async (ticketText) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/classify-ticket`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText
    })
  });
  const data = await response.json();
  return {
    category: data.category, // 'technical', 'billing', 'general'
    priority: data.priority,  // 'low', 'medium', 'high'
    department: data.department
  };
};
```

### AI Analytics

**Risk Prediction**
```javascript
// POST /api/ml/predict-risk
const predictUserRisk = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/predict-risk`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ userId })
  });
  const data = await response.json();
  return {
    riskScore: data.risk_score, // 0-1
    riskLevel: data.risk_level, // 'low', 'medium', 'high'
    factors: data.factors
  };
};
```

**Burnout Detection**
```javascript
// POST /api/ml/detect-burnout
const detectBurnout = async (userId) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/detect-burnout`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ userId })
  });
  const data = await response.json();
  return {
    burnoutScore: data.burnout_score,
    indicators: data.indicators,
    recommendations: data.recommendations
  };
};
```

**Anomaly Detection**
```javascript
// POST /api/ml/detect-anomaly
const detectAnomaly = async (userActivity) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/detect-anomaly`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userActivity.userId,
      loginTime: userActivity.loginTime,
      activityPattern: userActivity.pattern,
      location: userActivity.location
    })
  });
  const data = await response.json();
  return {
    isAnomaly: data.is_anomaly,
    anomalyScore: data.score,
    reason: data.reason
  };
};
```

**Project Delay Prediction**
```javascript
// POST /api/ml/predict-delay
const predictProjectDelay = async (projectId) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/predict-delay`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ projectId })
  });
  const data = await response.json();
  return {
    delayProbability: data.delay_probability,
    estimatedDelay: data.estimated_delay_days,
    riskFactors: data.risk_factors
  };
};
```

## Common Patterns

### Protected Route Component

```javascript
// frontend/src/components/ProtectedRoute.js
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const userRole = localStorage.getItem('role');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && userRole !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Kanban Board Implementation

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inprogress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/tasks?userId=${userId}`,
      {
        headers: { 'Authorization': `Bearer ${token}` }
      }
    );
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inprogress: data.filter(t => t.status === 'inprogress'),
      done: data.filter(t => t.status === 'done')
    });
  };

  const moveTask = async (taskId, newStatus) => {
    await updateTaskStatus(taskId, newStatus);
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} onMove={moveTask} targetStatus="todo" />
      <Column title="In Progress" tasks={tasks.inprogress} onMove={moveTask} targetStatus="inprogress" />
      <Column title="Done" tasks={tasks.done} onMove={moveTask} targetStatus="done" />
    </div>
  );
};
```

### Time Tracking Component

```javascript
// frontend/src/components/TimeTracker.js
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const [startTime, setStartTime] = useState(null);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const startTracking = () => {
    setStartTime(new Date());
    setIsRunning(true);
  };

  const stopTracking = async () => {
    setIsRunning(false);
    const endTime = new Date();
    const duration = Math.floor(seconds / 60); // minutes
    
    await trackTime(taskId, {
      startTime,
      endTime,
      duration
    });
    
    setSeconds(0);
    setStartTime(null);
  };

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      {!isRunning ? (
        <button onClick={startTracking}>Start</button>
      ) : (
        <button onClick={stopTracking}>Stop</button>
      )}
    </div>
  );
};
```

### Admin Dashboard with AI Insights

```javascript
// frontend/src/pages/AdminDashboard.js
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    // Fetch users
    const usersData = await getAllUsers();
    setUsers(usersData);

    // Fetch analytics
    const analyticsData = await fetchAnalytics();
    setAnalytics(analyticsData);

    // Check for AI alerts
    for (const user of usersData) {
      // Risk prediction
      const risk = await predictUserRisk(user._id);
      if (risk.riskLevel === 'high') {
        setAlerts(prev => [...prev, {
          type: 'risk',
          userId: user._id,
          userName: user.name,
          message: `High risk detected: ${risk.factors.join(', ')}`
        }]);
      }

      // Burnout detection
      const burnout = await detectBurnout(user._id);
      if (burnout.burnoutScore > 0.7) {
        setAlerts(prev => [...prev, {
          type: 'burnout',
          userId: user._id,
          userName: user.name,
          message: `Burnout risk: ${burnout.indicators.join(', ')}`
        }]);
      }
    }
  };

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`${process.env.REACT_APP_API_URL}/analytics`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    return await response.json();
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <section className="alerts">
        <h2>AI Alerts</h2>
        {alerts.map((alert, idx) => (
          <div key={idx} className={`alert alert-${alert.type}`}>
            <strong>{alert.userName}</strong>: {alert.message}
          </div>
        ))}
      </section>

      <section className="analytics">
        <h2>Organization Analytics</h2>
        {analytics && (
          <div className="stats-grid">
            <div className="stat">Total Users: {analytics.totalUsers}</div>
            <div className="stat">Active Tasks: {analytics.activeTasks}</div>
            <div className="stat">Open Tickets: {analytics.openTickets}</div>
            <div className="stat">Completion Rate: {analytics.completionRate}%</div>
          </div>
        )}
      </section>

      <section className="user-list">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.name}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.isActive ? 'Active' : 'Inactive'}</td>
                <td>
                  <button onClick={() => handleEdit(user._id)}>Edit</button>
                  <button onClick={() => handleDelete(user._id)}>Delete</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>
    </div>
  );
};
```

## Backend Middleware

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({ message: 'Access token required' });
  }

  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) {
      return res.status(403).json({ message: 'Invalid or expired token' });
    }
    req.user = user;
    next();
  });
};

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authenticateToken, requireAdmin };
```

### Audit Logging Middleware

```javascript
// backend/middleware/auditLog.js
const AuditLog = require('../models/AuditLog');

const auditLog = (action) => {
  return async (req, res, next) => {
    const originalSend = res.send;
    
    res.send = function(data) {
      res.send = originalSend;
      
      // Log the action
      AuditLog.create({
        userId: req.user?.id,
        action,
        endpoint: req.originalUrl,
        method: req.method,
        timestamp: new Date(),
        statusCode: res.statusCode,
        ipAddress: req.ip
      }).catch(err => console.error('Audit log error:', err));
      
      return res.send(data);
    };
    
    next();
  };
};

module.exports = auditLog;
```

## ML Service Implementation

### FastAPI ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Dict
import pickle
import numpy as np
from river import anomaly, tree

app = FastAPI(title="Enterprise Management ML Service")

# Load models
risk_model = None
ticket_classifier = None

class TicketRequest(BaseModel):
    text: str

class RiskRequest(BaseModel):
    userId: str

class BurnoutRequest(BaseModel):
    userId: str
    tasksCompleted: int
    averageWorkHours: float
    overtimeHours: float
    daysWorked: int

@app.on_event("startup")
async def load_models():
    global risk_model, ticket_classifier
    # Load pre-trained models
    try:
        with open(f"{os.getenv('MODEL_PATH')}/risk_model.pkl", 'rb') as f:
            risk_model = pickle.load(f)
        with open(f"{os.getenv('MODEL_PATH')}/ticket_classifier.pkl", 'rb') as f:
            ticket_classifier = pickle.load(f)
    except FileNotFoundError:
        print("Models not found, initializing new ones")
        risk_model = tree.HoeffdingTreeClassifier()
        ticket_classifier = tree.HoeffdingTreeClassifier()

@app.post("/classify-ticket")
async def classify_ticket(request: TicketRequest):
    """AI-based ticket classification"""
    features = extract_text_features(request.text)
    
    # Predict category
    category_map = {0: 'technical', 1: 'billing', 2: 'general'}
    priority_map = {0: 'low', 1: 'medium', 2: 'high'}
    
    category_pred = ticket_classifier.predict_one(features)
    
    # Determine priority based on keywords
    priority = determine_priority(request.text)
    department = route_to_department(category_pred)
    
    return {
        "category": category_map.get(category_pred, 'general'),
        "priority": priority,
        "department": department
    }

@app.post("/predict-risk")
async def predict_risk(request: RiskRequest):
    """Predict user risk score"""
    # Fetch user activity data from backend
    user_data = await fetch_user_data(request.userId)
    
    features = {
        'failed_logins': user_data.get('failedLogins', 0),
        'tasks_overdue': user_data.get('tasksOverdue', 0),
        'avg_response_time': user_data.get('avgResponseTime', 0),
        'policy_violations': user_data.get('policyViolations', 0)
    }
    
    risk_score = calculate_risk_score(features)
    risk_level = 'high' if risk_score > 0.7 else 'medium' if risk_score > 0.4 else 'low'
    
    factors = []
    if features['failed_logins'] > 3:
        factors.append('Multiple failed login attempts')
    if features['tasks_overdue'] > 5:
        factors.append('High number of overdue tasks')
    
    return {
        "risk_score": risk_score,
        "risk_level": risk_level,
        "factors": factors
    }

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutRequest):
    """Detect employee burnout risk"""
    burnout_score = calculate_burnout_score(
        request.tasksCompleted,
        request.averageWorkHours,
        request.overtimeHours,
        request.daysWorked
    )
    
    indicators = []
    recommendations = []
    
    if request.averageWorkHours > 50:
        indicators.append('Excessive work hours')
        recommendations.append('Reduce workload and encourage time off')
    
    if request.overtimeHours > 20:
        indicators.append('High overtime')
        recommendations.append('Limit overtime to 10 hours per week')
    
    if request.daysWorked > 15:
        indicators.append('No recent breaks')
        recommendations.append('Schedule mandatory rest days')
    
    return {
        "burnout_score": burnout_score,
        "indicators": indicators,
        "recommendations": recommendations
    }

def calculate_burnout_score(tasks, avg_hours, overtime, days_worked):
    """Calculate burnout score based on workload metrics"""
    score = 0.0
    
    # Normalize and weight factors
    score += min(avg_hours / 60, 1.0) * 0.3
    score += min(overtime / 30, 1.0) * 0.3
    score += min(days_worked / 20, 1.0) * 0.2
    score += min(tasks / 50, 1.0) * 0.2
    
    return round(score, 2)

def calculate_risk_score(features):
    """Calculate overall risk score"""
    weights = {
        'failed_logins': 0.3,
        'tasks_overdue': 0.25,
        'avg_response_time': 0.2,
        'policy_violations': 0.25
    }
    
    score = sum(
        min(features[k] / 10, 1.0) * weights[k]
        for k in features
    )
    
    return round(score, 2)
```

## Configuration

### MongoDB Schema Examples

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  isActive: { type: Boolean, default: true },
  department: String,
  joinDate: { type: Date, default: Date.now },
  lastLogin: Date,
  failedLoginAttempts: { type: Number, default: 0 }
}, { timestamps: true });

module.exports = mongoose.model('User', userSchema);
```

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['todo', 'inprogress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  dueDate: Date,
  timeTracked: [{
    startTime: Date,
    endTime: Date,
    duration: Number // minutes
  }],
  tags: [String]
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

## Troubleshooting

### Common Issues

**JWT Token Expired**
```javascript
// Frontend: Implement token refresh
const refreshToken = async () => {
  try {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/refresh`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ 
        refreshToken: localStorage.getItem('refreshToken') 
      })
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

**MongoDB Connection Issues**
```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

**ML Service Not Responding**
```bash
# Check if service is running
curl http://localhost:8000/docs

# Restart with verbose logging
uvicorn main:app --reload --log-level debug
```

**CORS Issues**
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

**Model Loading Errors**
```python
# ml-service/main.py
import os

@app.on_event("startup")
async def load_models():
    model_path = os.getenv('MODEL_PATH', './models')
    os.makedirs(model_path, exist_ok=True)
    
    try:
        # Load models
        pass
    except Exception as e:
        print(f"Model loading error: {e}")
        # Initialize with defaults
```

## Testing

### API Testing Examples

```javascript
// test/api.test.js
const axios = require('axios');

const API_URL = 'http://localhost:5000/api';
let authToken;

describe('Authentication', () => {
  it('should register new user', async () => {
