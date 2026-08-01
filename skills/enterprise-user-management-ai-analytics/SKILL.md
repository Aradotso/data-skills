---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "help me build a user management dashboard"
  - "set up AI-powered task and ticket management"
  - "create an enterprise admin system with analytics"
  - "implement risk detection and burnout analysis"
  - "build a kanban board with time tracking"
  - "add AI assistant for user management"
  - "integrate anomaly detection in user system"
  - "create role-based access control system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack application for managing enterprise users, tasks, and support tickets with integrated AI analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

- **User Management**: Role-based access control with admin and user dashboards
- **Task Tracking**: Kanban board (To Do → In Progress → Done) with time tracking
- **Ticket System**: Support ticket management with AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Authentication**: JWT-based secure login and session management
- **Monitoring**: Audit logs, performance insights, and suspicious activity alerts

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

Create `.env` file:

```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
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

Create `ml-service/.env`:

```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
ENABLE_TRAINING=true
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

Create `frontend/.env`:

```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Runs at http://localhost:3000
```

## Key API Endpoints

### Authentication

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
      role: 'user' // or 'admin'
    })
  });
  return await response.json();
};

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

### User Management

```javascript
// GET /api/users - Admin only
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return await response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};

// DELETE /api/users/:id - Admin only
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

### Task Management

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
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
      priority: 'medium', // low, medium, high
      status: 'todo', // todo, in_progress, done
      dueDate: taskData.dueDate
    })
  });
  return await response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, newStatus, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return await response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

### Support Tickets

```javascript
// POST /api/tickets - Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      category: ticketData.category, // technical, billing, general
      priority: 'medium'
    })
  });
  return await response.json();
};

// GET /api/tickets - Get all tickets (admin) or user tickets
const getTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};

// PATCH /api/tickets/:id - Update ticket
const updateTicket = async (ticketId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};
```

## AI/ML Service Integration

### Risk Detection

```javascript
// POST /api/ml/risk-detection
const detectUserRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.userId,
      loginFrequency: userData.loginFrequency,
      taskCompletionRate: userData.taskCompletionRate,
      failedLoginAttempts: userData.failedLoginAttempts,
      accessPatterns: userData.accessPatterns
    })
  });
  const result = await response.json();
  // Returns: { riskScore: 0.75, riskLevel: "high", factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/anomaly-detection
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: activityData.userId,
      timestamp: activityData.timestamp,
      activityType: activityData.activityType,
      ipAddress: activityData.ipAddress,
      deviceInfo: activityData.deviceInfo,
      location: activityData.location
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, confidence: 0.92, reason: "..." }
  return result;
};
```

### Burnout Analysis

```javascript
// POST /api/ml/burnout-analysis
const analyzeBurnout = async (userMetrics) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-analysis', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userMetrics.userId,
      workingHours: userMetrics.workingHours,
      tasksCompleted: userMetrics.tasksCompleted,
      missedDeadlines: userMetrics.missedDeadlines,
      overtimeHours: userMetrics.overtimeHours,
      weekendWork: userMetrics.weekendWork
    })
  });
  const result = await response.json();
  // Returns: { burnoutScore: 0.68, level: "moderate", recommendations: [...] }
  return result;
};
```

### Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      subject: ticketText.subject,
      description: ticketText.description
    })
  });
  const result = await response.json();
  // Returns: { category: "technical", priority: "high", suggestedAssignee: "..." }
  return result;
};
```

### Predictive Insights

```javascript
// POST /api/ml/predict-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.projectId,
      tasksRemaining: projectData.tasksRemaining,
      avgCompletionTime: projectData.avgCompletionTime,
      teamSize: projectData.teamSize,
      deadline: projectData.deadline
    })
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.45, estimatedDelay: "3 days", factors: [...] }
  return result;
};
```

## Frontend Component Patterns

### Protected Route with Authentication

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  
  if (!token) {
    return <Navigate to="/login" replace />;
  }
  
  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" replace />;
  }
  
  return children;
};

// Usage in App.js
<Route path="/admin" element={
  <ProtectedRoute requiredRole="admin">
    <AdminDashboard />
  </ProtectedRoute>
} />
```

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    fetchTasks();
  }, []);
  
  const fetchTasks = async () => {
    const response = await fetch('http://localhost:5000/api/tasks/user/me', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      in_progress: data.filter(t => t.status === 'in_progress'),
      done: data.filter(t => t.status === 'done')
    });
  };
  
  const moveTask = async (taskId, newStatus) => {
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };
  
  return (
    <div className="kanban-board">
      {['todo', 'in_progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select onChange={(e) => moveTask(task._id, e.target.value)} value={status}>
                <option value="todo">To Do</option>
                <option value="in_progress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [isTracking, setIsTracking] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    let interval;
    if (isTracking) {
      interval = setInterval(() => {
        setElapsedTime(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isTracking]);
  
  const startTracking = async () => {
    setIsTracking(true);
    await fetch(`http://localhost:5000/api/tasks/${taskId}/start-timer`, {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${token}` }
    });
  };
  
  const stopTracking = async () => {
    setIsTracking(false);
    await fetch(`http://localhost:5000/api/tasks/${taskId}/stop-timer`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ timeSpent: elapsedTime })
    });
  };
  
  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };
  
  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(elapsedTime)}</div>
      <button onClick={isTracking ? stopTracking : startTracking}>
        {isTracking ? 'Stop' : 'Start'}
      </button>
    </div>
  );
};

export default TimeTracker;
```

## Database Models

### User Model (MongoDB)

```javascript
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive', 'suspended'], default: 'active' },
  lastLogin: Date,
  failedLoginAttempts: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date
});

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in_progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in seconds
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date,
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  ticketNumber: { type: String, unique: true },
  subject: { type: String, required: true },
  description: { type: String, required: true },
  category: { type: String, enum: ['technical', 'billing', 'general'], default: 'general' },
  priority: { type: String, enum: ['low', 'medium', 'high', 'urgent'], default: 'medium' },
  status: { type: String, enum: ['open', 'in_progress', 'resolved', 'closed'], default: 'open' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: {
    category: String,
    confidence: Number,
    suggestedPriority: String
  },
  comments: [{
    user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
    text: String,
    timestamp: { type: Date, default: Date.now }
  }],
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date,
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Configuration

### Backend Configuration

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### JWT Middleware

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

### ML Service Configuration

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sklearn.ensemble import RandomForestClassifier, IsolationForest
import numpy as np
import joblib
import os

app = FastAPI()

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class RiskDetectionInput(BaseModel):
    userId: str
    loginFrequency: float
    taskCompletionRate: float
    failedLoginAttempts: int
    accessPatterns: list

class AnomalyDetectionInput(BaseModel):
    userId: str
    timestamp: str
    activityType: str
    ipAddress: str
    deviceInfo: str
    location: str

class BurnoutAnalysisInput(BaseModel):
    userId: str
    workingHours: float
    tasksCompleted: int
    missedDeadlines: int
    overtimeHours: float
    weekendWork: int

@app.post("/api/ml/risk-detection")
async def detect_risk(data: RiskDetectionInput):
    # Risk scoring logic
    risk_score = 0.0
    factors = []
    
    if data.failedLoginAttempts > 3:
        risk_score += 0.3
        factors.append("Multiple failed login attempts")
    
    if data.taskCompletionRate < 0.5:
        risk_score += 0.2
        factors.append("Low task completion rate")
    
    if data.loginFrequency < 0.3:
        risk_score += 0.15
        factors.append("Infrequent login activity")
    
    risk_level = "low" if risk_score < 0.3 else "medium" if risk_score < 0.7 else "high"
    
    return {
        "riskScore": round(risk_score, 2),
        "riskLevel": risk_level,
        "factors": factors
    }

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(data: AnomalyDetectionInput):
    # Simplified anomaly detection
    is_anomaly = False
    confidence = 0.0
    reason = ""
    
    # Example: Detect unusual access patterns
    if "unknown" in data.location.lower() or "vpn" in data.ipAddress.lower():
        is_anomaly = True
        confidence = 0.85
        reason = "Unusual access location or IP"
    
    return {
        "isAnomaly": is_anomaly,
        "confidence": confidence,
        "reason": reason
    }

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(data: BurnoutAnalysisInput):
    # Burnout scoring
    burnout_score = 0.0
    recommendations = []
    
    if data.workingHours > 50:
        burnout_score += 0.3
        recommendations.append("Reduce working hours to recommended limit")
    
    if data.overtimeHours > 10:
        burnout_score += 0.2
        recommendations.append("Minimize overtime work")
    
    if data.weekendWork > 2:
        burnout_score += 0.15
        recommendations.append("Avoid weekend work")
    
    if data.missedDeadlines > 3:
        burnout_score += 0.25
        recommendations.append("Review task allocation and deadlines")
    
    level = "low" if burnout_score < 0.3 else "moderate" if burnout_score < 0.6 else "high"
    
    return {
        "burnoutScore": round(burnout_score, 2),
        "level": level,
        "recommendations": recommendations
    }

@app.post("/api/ml/classify-ticket")
async def classify_ticket(data: dict):
    subject = data.get('subject', '').lower()
    description = data.get('description', '').lower()
    
    # Simple keyword-based classification
    category = "general"
    priority = "medium"
    
    if any(word in subject + description for word in ['bug', 'error', 'crash', 'not working']):
        category = "technical"
        priority = "high"
    elif any(word in subject + description for word in ['payment', 'invoice', 'billing']):
        category = "billing"
        priority = "medium"
    
    return {
        "category": category,
        "priority": priority,
        "suggestedAssignee": "tech-support" if category == "technical" else "general-support"
    }
```

## Common Patterns

### API Request Wrapper with Error Handling

```javascript
// frontend/utils/api.js
const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';

export const apiRequest = async (endpoint, options = {}) => {
  const token = localStorage.getItem('token');
  
  const config = {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...(token && { 'Authorization': `Bearer ${token}` }),
      ...options.headers
    }
  };
  
  try {
    const response = await fetch(`${API_BASE_URL}${endpoint}`, config);
    const data = await response.json();
    
    if (!response.ok) {
      throw new Error(data.error || 'Request failed');
    }
    
    return data;
  } catch (error) {
    console.error('API Error:', error);
    throw error;
  }
};

// Usage
import { apiRequest } from './utils/api';

const fetchUserData = async () => {
  try {
    const data = await apiRequest('/api/users/me');
    return data;
  } catch (error) {
    console.error('Failed to fetch user data:', error);
  }
};
```

### Real-time Notifications Hook

```javascript
// frontend/hooks/useNotifications.js
import { useState, useEffect } from 'react';

export const useNotifications = () => {
  const [notifications, setNotifications] = useState([]);
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    const fetchNotifications = async () => {
      const response = await fetch('http://localhost:5000/api/notifications', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setNotifications(data);
    };
    
    fetchNotifications();
    const interval = setInterval(fetchNotifications, 30000); // Poll every 30s
    
    return () => clearInterval(interval);
  }, [token]);
  
  const markAsRead = async (notificationId) => {
    await fetch(`http://localhost:5000/api/notifications/${notificationId}/read`, {
      method: 'PATCH',
      headers: { 'Authorization': `Bearer ${token}` }
    });
    setNotifications(prev => 
      prev.map(n => n._id === notificationId ? { ...n, read: true } : n)
    );
  };
  
  return { notifications, markAsRead };
};
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check if MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# View MongoDB logs
tail -f /var/log/mongodb/mongod.log
```

### JWT Token Expiration

```javascript
// Add token refresh logic
const refreshToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  const response = await fetch('http://localhost:5000/api/auth/refresh', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken })
  });
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data.token;
};
```

### ML Service Not Responding

```bash
# Check if ML service is running
curl http://localhost:8000/docs

# View logs
cd ml-service
uvicorn main:app --reload --log-level debug

# Check Python dependencies
pip list | grep -E 'fastapi|scikit-learn|river'
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Performance Optimization

```javascript
// Enable caching for frequently accessed data
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 600 }); // 10 minutes

app.get('/api/analytics/dashboard', authMiddleware, async (req, res) => {
  const cacheKey = `dashboard_${req.user._id}`;
  const cached = cache.get(cacheKey);
  
  if (cached) {
    return res.json(cached);
  }
  
  const data = await generateDashboardData(req.user._id);
  cache.set(cacheKey, data);
  res.json(data);
});
```

### Database Indexing

```javascript
// Add indexes for better query performance
userSchema.index({ email: 1 });
userSchema.index({ role: 1, status: 1 });
taskSchema.index({ assignedTo
