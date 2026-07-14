---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "help me set up the enterprise user management system"
  - "how do I implement AI analytics for user management"
  - "integrate user management with AI predictions"
  - "build a task tracking system with burnout detection"
  - "create a support ticket system with AI classification"
  - "set up JWT authentication for user management"
  - "implement Kanban board with time tracking"
  - "configure AI-based anomaly detection for users"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

The Enterprise User Management System is a full-stack JavaScript application that provides comprehensive user, task, and ticket management with integrated AI/ML capabilities. It features:

- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards, time tracking, progress monitoring
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, user monitoring

The system combines a React frontend, Node.js backend, and FastAPI ML service with MongoDB for data persistence.

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

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
EOF

npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
EOF

uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

npm start
```

## Architecture Overview

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   React     │─────▶│   Node.js   │─────▶│   MongoDB   │
│  Frontend   │      │   Backend   │      │  Database   │
└─────────────┘      └─────────────┘      └─────────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │   FastAPI   │
                     │  ML Service │
                     └─────────────┘
```

## Key API Endpoints

### Authentication

```javascript
// Register user
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
// Response: { token: "jwt_token", user: {...} }
```

### User Management

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { Authorization: "Bearer <token>" }

// Update user
PUT /api/users/:id
{
  "name": "Updated Name",
  "role": "admin"
}

// Delete user
DELETE /api/users/:id
```

### Task Management

```javascript
// Create task
POST /api/tasks
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "userId",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-12-31"
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:id/time
{
  "timeSpent": 3600 // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error",
  "priority": "medium",
  "category": "technical"
}

// AI Classification (automatic)
GET /api/tickets/:id/classify
// Returns: { category: "technical", priority: "high", suggestedAssignee: "userId" }
```

### AI Analytics

```javascript
// Risk prediction
POST /api/ai/risk-prediction
{
  "userId": "user123",
  "features": {
    "taskCompletionRate": 0.65,
    "averageDelay": 2.5,
    "ticketCount": 5
  }
}
// Response: { riskLevel: "medium", score: 0.62, factors: [...] }

// Anomaly detection
POST /api/ai/anomaly-detection
{
  "userId": "user123",
  "behavior": {
    "loginTime": "03:00",
    "accessPattern": "unusual",
    "dataAccess": "high"
  }
}

// Burnout detection
GET /api/ai/burnout/:userId
// Response: { burnoutScore: 0.75, recommendation: "reduce workload" }

// Project insights
POST /api/ai/project-insights
{
  "projectId": "proj123",
  "tasks": [...],
  "timeline": "2026-12-31"
}
```

## Frontend Integration Examples

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
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
      const response = await axios.get(`${API_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', response.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
    setUser(response.data.user);
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return { user, loading, login, logout };
};
```

### Task Board Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks?userId=${userId}`);
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
      await axios.patch(`${API_URL}/api/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = (e, status) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, status);
  };

  return (
    <div className="task-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div
          key={status}
          className="task-column"
          onDragOver={(e) => e.preventDefault()}
          onDrop={(e) => handleDrop(e, status)}
        >
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div
              key={task._id}
              className="task-card"
              draggable
              onDragStart={(e) => handleDragStart(e, task._id)}
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-${task.priority}`}>
                {task.priority}
              </span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};
```

### Time Tracker Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const startTimer = () => setIsRunning(true);
  
  const stopTimer = async () => {
    setIsRunning(false);
    if (seconds > 0) {
      try {
        await axios.post(`${API_URL}/api/tasks/${taskId}/time`, {
          timeSpent: seconds
        });
        setSeconds(0);
      } catch (error) {
        console.error('Failed to log time:', error);
      }
    }
  };

  const formatTime = (secs) => {
    const hrs = Math.floor(secs / 3600);
    const mins = Math.floor((secs % 3600) / 60);
    const s = secs % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <div className="timer-controls">
        {!isRunning ? (
          <button onClick={startTimer}>Start</button>
        ) : (
          <button onClick={stopTimer}>Stop</button>
        )}
      </div>
    </div>
  );
};
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutScore: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      const [risk, burnout, anomalies] = await Promise.all([
        axios.get(`${API_URL}/api/ai/risk/${userId}`),
        axios.get(`${API_URL}/api/ai/burnout/${userId}`),
        axios.get(`${API_URL}/api/ai/anomalies/${userId}`)
      ]);

      setAnalytics({
        riskScore: risk.data,
        burnoutScore: burnout.data,
        anomalies: anomalies.data
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <div className="metric-card">
        <h3>Risk Score</h3>
        <div className={`score score-${analytics.riskScore?.level}`}>
          {analytics.riskScore?.score?.toFixed(2) || 'N/A'}
        </div>
        <p>{analytics.riskScore?.recommendation}</p>
      </div>

      <div className="metric-card">
        <h3>Burnout Detection</h3>
        <div className="burnout-meter">
          <div
            className="burnout-level"
            style={{ width: `${(analytics.burnoutScore?.score || 0) * 100}%` }}
          />
        </div>
        <p>{analytics.burnoutScore?.recommendation}</p>
      </div>

      <div className="anomalies-list">
        <h3>Recent Anomalies</h3>
        {analytics.anomalies.map(anomaly => (
          <div key={anomaly.id} className="anomaly-item">
            <span className="anomaly-type">{anomaly.type}</span>
            <span className="anomaly-time">{new Date(anomaly.timestamp).toLocaleString()}</span>
            <p>{anomaly.description}</p>
          </div>
        ))}
      </div>
    </div>
  );
};
```

## Backend Implementation Patterns

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        message: `User role ${req.user.role} is not authorized to access this route`
      });
    }
    next();
  };
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });

    // Trigger AI analysis for project insights
    if (req.body.projectId) {
      await axios.post(`${process.env.ML_SERVICE_URL}/api/update-project`, {
        projectId: req.body.projectId,
        taskId: task._id
      });
    }

    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    task.status = req.body.status;
    if (req.body.status === 'done') {
      task.completedAt = new Date();
    }

    await task.save();

    // Log time spent
    if (req.body.timeSpent) {
      task.timeSpent = (task.timeSpent || 0) + req.body.timeSpent;
      await task.save();
    }

    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({
      $or: [
        { assignedTo: req.user.id },
        { createdBy: req.user.id }
      ]
    }).populate('assignedTo', 'name email');

    res.json(tasks);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};
```

### Ticket Controller with AI Integration

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

exports.createTicket = async (req, res) => {
  try {
    const ticket = await Ticket.create({
      ...req.body,
      createdBy: req.user.id,
      status: 'open'
    });

    // AI Classification
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/classify-ticket`,
        {
          title: ticket.title,
          description: ticket.description
        }
      );

      ticket.category = mlResponse.data.category;
      ticket.aiPriority = mlResponse.data.priority;
      ticket.suggestedAssignee = mlResponse.data.assignee;
      await ticket.save();
    } catch (mlError) {
      console.error('ML classification failed:', mlError);
    }

    res.status(201).json(ticket);
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
import joblib
import numpy as np
from typing import List, Dict
import os

app = FastAPI()

MODEL_PATH = os.getenv('MODEL_PATH', './models')

# Load models
try:
    risk_model = joblib.load(f'{MODEL_PATH}/risk_model.pkl')
    ticket_classifier = joblib.load(f'{MODEL_PATH}/ticket_classifier.pkl')
    anomaly_detector = joblib.load(f'{MODEL_PATH}/anomaly_detector.pkl')
except:
    print("Models not found, using default configurations")

class RiskPredictionRequest(BaseModel):
    userId: str
    taskCompletionRate: float
    averageDelay: float
    ticketCount: int
    loginFrequency: float

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class AnomalyDetectionRequest(BaseModel):
    userId: str
    loginTime: str
    accessPattern: str
    dataVolume: int

@app.post("/api/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        # Simple keyword-based classification
        text = f"{request.title} {request.description}".lower()
        
        categories = {
            'technical': ['error', 'bug', 'crash', 'not working'],
            'access': ['login', 'password', 'access', 'permission'],
            'feature': ['feature', 'request', 'add', 'improve']
        }
        
        category = 'general'
        for cat, keywords in categories.items():
            if any(kw in text for kw in keywords):
                category = cat
                break
        
        priority = 'medium'
        if any(word in text for word in ['urgent', 'critical', 'crash']):
            priority = 'high'
        elif any(word in text for word in ['minor', 'suggestion']):
            priority = 'low'
        
        return {
            'category': category,
            'priority': priority,
            'confidence': 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/risk-prediction")
async def predict_risk(request: RiskPredictionRequest):
    try:
        features = np.array([[
            request.taskCompletionRate,
            request.averageDelay,
            request.ticketCount,
            request.loginFrequency
        ]])
        
        # Calculate risk score
        risk_score = (
            (1 - request.taskCompletionRate) * 0.4 +
            min(request.averageDelay / 10, 1) * 0.3 +
            min(request.ticketCount / 20, 1) * 0.2 +
            (1 - min(request.loginFrequency / 5, 1)) * 0.1
        )
        
        level = 'low' if risk_score < 0.3 else 'medium' if risk_score < 0.7 else 'high'
        
        return {
            'score': float(risk_score),
            'level': level,
            'factors': {
                'completionRate': request.taskCompletionRate,
                'avgDelay': request.averageDelay,
                'ticketCount': request.ticketCount
            },
            'recommendation': f"Risk level is {level}. {'Consider workload adjustment.' if level == 'high' else 'Monitor closely.' if level == 'medium' else 'Continue current pace.'}"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/anomaly-detection")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        anomalies = []
        
        # Check unusual login time
        hour = int(request.loginTime.split(':')[0])
        if hour < 6 or hour > 22:
            anomalies.append({
                'type': 'unusual_login_time',
                'severity': 'medium',
                'description': f'Login at unusual hour: {request.loginTime}'
            })
        
        # Check access pattern
        if request.accessPattern == 'unusual':
            anomalies.append({
                'type': 'unusual_access_pattern',
                'severity': 'high',
                'description': 'Detected unusual access pattern'
            })
        
        # Check data volume
        if request.dataVolume > 10000:
            anomalies.append({
                'type': 'high_data_access',
                'severity': 'high',
                'description': f'Unusually high data access: {request.dataVolume} records'
            })
        
        return {
            'isAnomaly': len(anomalies) > 0,
            'anomalies': anomalies,
            'userId': request.userId
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/burnout/{user_id}")
async def detect_burnout(user_id: str):
    # This would typically fetch user data from backend
    # For now, returning sample calculation
    return {
        'score': 0.65,
        'level': 'moderate',
        'factors': {
            'workload': 'high',
            'overtime': 'frequent',
            'taskCompletion': 'declining'
        },
        'recommendation': 'Consider reducing task load and scheduling breaks'
    }
```

## Configuration

### Environment Variables

**Backend (.env)**:
```
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
FRONTEND_URL=http://localhost:3000
```

**Frontend (.env)**:
```
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

**ML Service (.env)**:
```
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Common Patterns

### Protected Routes

```javascript
// frontend/src/App.js
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { useAuth } from './hooks/useAuth';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  if (!user) return <Navigate to="/login" />;
  if (adminOnly && user.role !== 'admin') return <Navigate to="/dashboard" />;

  return children;
};

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route
          path="/dashboard"
          element={
            <ProtectedRoute>
              <Dashboard />
            </ProtectedRoute>
          }
        />
        <Route
          path="/admin"
          element={
            <ProtectedRoute adminOnly>
              <AdminPanel />
            </ProtectedRoute>
          }
        />
      </Routes>
    </BrowserRouter>
  );
}
```

### Real-time Notifications

```javascript
// frontend/src/hooks/useNotifications.js
import { useEffect, useState } from 'react';
import io from 'socket.io-client';

export const useNotifications = (userId) => {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    const socket = io(process.env.REACT_APP_API_URL);

    socket.on('connect', () => {
      socket.emit('join', userId);
    });

    socket.on('notification', (notification) => {
      setNotifications(prev => [notification, ...prev]);
    });

    return () => socket.disconnect();
  }, [userId]);

  return notifications;
};
```

## Troubleshooting

### Authentication Issues

**Problem**: JWT token expired
```javascript
// Add axios interceptor to handle token refresh
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### MongoDB Connection Issues

**Problem**: Cannot connect to MongoDB
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
    console.log('MongoDB Connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};
```

### ML Service Not Responding

**Problem**: ML predictions failing
```javascript
// Add fallback in backend
const getPrediction = async (data) => {
  try {
    const response = await axios.post(
      `${process.env.ML_SERVICE_URL}/api/risk-prediction`,
      data,
      { timeout: 5000 }
    );
    return response.data;
  } catch (error) {
    console.error('ML service error:', error);
    // Return default/cached prediction
    return {
      score: 0.5,
      level: 'medium',
      recommendation: 'ML service unavailable, showing default prediction'
    };
  }
};
```

### CORS Issues

**Problem**: Frontend cannot access backend
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL,
  credentials: true
}));
```

### Performance Optimization

**Problem**: Slow dashboard loading
```javascript
// Implement pagination and caching
exports.getUsers = async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 10;
  const skip = (page - 1) * limit;

  const users = await User.find()
    .select('-password')
    .limit(limit)
    .skip(skip)
    .lean(); // Use lean() for faster queries

  const total = await User.countDocuments();

  res.json({
    users,
    currentPage: page,
    totalPages: Math.ceil(total / limit),
    total
  });
};
```
