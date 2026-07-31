---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and workforce insights
triggers:
  - "help me build a user management dashboard with AI analytics"
  - "how do I integrate AI risk detection into my user management system"
  - "set up enterprise task management with burnout detection"
  - "implement JWT authentication with role-based access control"
  - "create a ticket management system with AI classification"
  - "build a kanban board with time tracking features"
  - "add anomaly detection to user activity monitoring"
  - "develop admin dashboard with user analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user management, task tracking, and support ticket systems with AI-powered insights. It provides risk detection, anomaly detection, burnout analysis, and predictive project insights to help organizations improve productivity and automate workflows.

**Key Components:**
- **Frontend:** React.js dashboard for users and admins
- **Backend:** Node.js REST API with JWT authentication
- **ML Service:** FastAPI service with scikit-learn and River for real-time AI analytics
- **Database:** MongoDB for data persistence

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Setup Backend
cd backend
npm install
cp .env.example .env  # Configure environment variables
npm start  # Runs on http://localhost:5000

# Setup ML Service
cd ../ml-service
pip install -r requirements.txt
uvicorn main:app --reload  # Runs on http://localhost:8000

# Setup Frontend
cd ../frontend
npm install
cp .env.example .env  # Configure API endpoints
npm start  # Runs on http://localhost:3000
```

### Environment Variables

**Backend (.env):**
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRES_IN=7d
PORT=5000
ML_SERVICE_URL=http://localhost:8000
```

**Frontend (.env):**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

**ML Service (.env):**
```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Backend API Reference

### Authentication

```javascript
// User Login
const axios = require('axios');

const login = async (email, password) => {
  const response = await axios.post('http://localhost:5000/api/auth/login', {
    email,
    password
  });
  
  const { token, user } = response.data;
  // Store token for subsequent requests
  localStorage.setItem('token', token);
  return user;
};

// Register New User
const register = async (userData) => {
  const response = await axios.post('http://localhost:5000/api/auth/register', {
    name: userData.name,
    email: userData.email,
    password: userData.password,
    role: userData.role || 'user'
  });
  return response.data;
};
```

### User Management (Admin Only)

```javascript
// Get all users
const getUsers = async (token) => {
  const response = await axios.get('http://localhost:5000/api/users', {
    headers: { Authorization: `Bearer ${token}` }
  });
  return response.data;
};

// Update user
const updateUser = async (userId, updates, token) => {
  const response = await axios.put(
    `http://localhost:5000/api/users/${userId}`,
    updates,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Delete user
const deleteUser = async (userId, token) => {
  await axios.delete(`http://localhost:5000/api/users/${userId}`, {
    headers: { Authorization: `Bearer ${token}` }
  });
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData, token) => {
  const response = await axios.post(
    'http://localhost:5000/api/tasks',
    {
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'inprogress', 'done'
    },
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await axios.get(
    `http://localhost:5000/api/tasks/user/${userId}`,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await axios.patch(
    `http://localhost:5000/api/tasks/${taskId}/status`,
    { status },
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Track time on task
const trackTime = async (taskId, duration, token) => {
  const response = await axios.post(
    `http://localhost:5000/api/tasks/${taskId}/time`,
    { duration }, // Duration in seconds
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};
```

### Support Tickets

```javascript
// Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await axios.post(
    'http://localhost:5000/api/tickets',
    {
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    },
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Get tickets (admin or assigned user)
const getTickets = async (filters, token) => {
  const response = await axios.get('http://localhost:5000/api/tickets', {
    params: filters, // { status: 'open', priority: 'high' }
    headers: { Authorization: `Bearer ${token}` }
  });
  return response.data;
};

// Update ticket
const updateTicket = async (ticketId, updates, token) => {
  const response = await axios.put(
    `http://localhost:5000/api/tickets/${ticketId}`,
    updates,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};
```

## ML Service API Reference

### AI-Powered Analytics

```javascript
// Risk prediction based on user behavior
const predictRisk = async (userData) => {
  const response = await axios.post('http://localhost:8000/api/ai/risk-prediction', {
    userId: userData.userId,
    taskCompletionRate: userData.taskCompletionRate,
    averageTaskDelay: userData.averageTaskDelay,
    ticketCount: userData.ticketCount,
    loginFrequency: userData.loginFrequency
  });
  
  return response.data; // { riskLevel: 'high', confidence: 0.85, factors: [...] }
};

// Anomaly detection for security
const detectAnomaly = async (activityData) => {
  const response = await axios.post('http://localhost:8000/api/ai/anomaly-detection', {
    userId: activityData.userId,
    loginTime: activityData.loginTime,
    ipAddress: activityData.ipAddress,
    location: activityData.location,
    actions: activityData.actions
  });
  
  return response.data; // { isAnomaly: true, score: 0.92, reason: '...' }
};

// Burnout detection
const detectBurnout = async (workloadData) => {
  const response = await axios.post('http://localhost:8000/api/ai/burnout-detection', {
    userId: workloadData.userId,
    tasksAssigned: workloadData.tasksAssigned,
    overtimeHours: workloadData.overtimeHours,
    taskCompletionTime: workloadData.taskCompletionTime,
    stressIndicators: workloadData.stressIndicators
  });
  
  return response.data; // { burnoutRisk: 'high', recommendations: [...] }
};

// Predictive project insights
const predictProjectDelay = async (projectData) => {
  const response = await axios.post('http://localhost:8000/api/ai/project-prediction', {
    projectId: projectData.projectId,
    tasksRemaining: projectData.tasksRemaining,
    averageCompletionRate: projectData.averageCompletionRate,
    teamSize: projectData.teamSize,
    deadline: projectData.deadline
  });
  
  return response.data; // { delayProbability: 0.75, estimatedCompletion: '2026-05-20' }
};

// AI ticket classification
const classifyTicket = async (ticketText) => {
  const response = await axios.post('http://localhost:8000/api/ai/classify-ticket', {
    subject: ticketText.subject,
    description: ticketText.description
  });
  
  return response.data; // { category: 'technical', priority: 'high', suggestedAssignee: 'userId' }
};
```

## Frontend React Components

### Protected Route with JWT

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    fetchTasks();
  }, [userId]);
  
  const fetchTasks = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    
    const grouped = {
      todo: response.data.filter(t => t.status === 'todo'),
      inprogress: response.data.filter(t => t.status === 'inprogress'),
      done: response.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };
  
  const moveTask = async (taskId, newStatus) => {
    await axios.patch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
      { status: newStatus },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    fetchTasks();
  };
  
  return (
    <div className="kanban-board">
      {['todo', 'inprogress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status}
                onChange={(e) => moveTask(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="inprogress">In Progress</option>
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
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);
  
  const handleStop = async () => {
    setIsRunning(false);
    await axios.post(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`,
      { duration: seconds },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setSeconds(0);
  };
  
  const formatTime = (sec) => {
    const h = Math.floor(sec / 3600);
    const m = Math.floor((sec % 3600) / 60);
    const s = sec % 60;
    return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };
  
  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={handleStop} disabled={seconds === 0}>
        Stop & Save
      </button>
    </div>
  );
};

export default TimeTracker;
```

### Admin Analytics Dashboard

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({});
  const [riskUsers, setRiskUsers] = useState([]);
  const token = localStorage.getItem('token');
  
  useEffect(() => {
    fetchAnalytics();
    fetchRiskUsers();
  }, []);
  
  const fetchAnalytics = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/admin/analytics`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setAnalytics(response.data);
  };
  
  const fetchRiskUsers = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/admin/risk-users`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setRiskUsers(response.data);
  };
  
  return (
    <div className="admin-dashboard">
      <div className="metrics">
        <div className="metric-card">
          <h3>Total Users</h3>
          <p>{analytics.totalUsers}</p>
        </div>
        <div className="metric-card">
          <h3>Active Tasks</h3>
          <p>{analytics.activeTasks}</p>
        </div>
        <div className="metric-card">
          <h3>Open Tickets</h3>
          <p>{analytics.openTickets}</p>
        </div>
      </div>
      
      <div className="risk-alerts">
        <h2>High-Risk Users</h2>
        {riskUsers.map(user => (
          <div key={user.userId} className="risk-card">
            <h4>{user.name}</h4>
            <p>Risk Level: {user.riskLevel}</p>
            <p>Reason: {user.reason}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Python ML Service Implementation

### Risk Prediction Model

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
import os

app = FastAPI()

# Load or initialize model
MODEL_PATH = os.getenv('MODEL_PATH', './models')
risk_model = None

try:
    risk_model = joblib.load(f'{MODEL_PATH}/risk_model.pkl')
except:
    # Initialize new model if not exists
    risk_model = RandomForestClassifier(n_estimators=100, random_state=42)

class RiskPredictionRequest(BaseModel):
    userId: str
    taskCompletionRate: float
    averageTaskDelay: float
    ticketCount: int
    loginFrequency: float

@app.post('/api/ai/risk-prediction')
async def predict_risk(request: RiskPredictionRequest):
    features = np.array([[
        request.taskCompletionRate,
        request.averageTaskDelay,
        request.ticketCount,
        request.loginFrequency
    ]])
    
    # If model is trained, predict
    if hasattr(risk_model, 'classes_'):
        prediction = risk_model.predict(features)[0]
        confidence = risk_model.predict_proba(features)[0].max()
        
        # Get feature importance
        importance = risk_model.feature_importances_
        factors = [
            {'factor': 'Task Completion Rate', 'importance': float(importance[0])},
            {'factor': 'Average Task Delay', 'importance': float(importance[1])},
            {'factor': 'Ticket Count', 'importance': float(importance[2])},
            {'factor': 'Login Frequency', 'importance': float(importance[3])}
        ]
        
        return {
            'riskLevel': prediction,
            'confidence': float(confidence),
            'factors': sorted(factors, key=lambda x: x['importance'], reverse=True)
        }
    else:
        # Return heuristic-based risk if model not trained
        risk_score = (
            (1 - request.taskCompletionRate) * 0.4 +
            min(request.averageTaskDelay / 10, 1) * 0.3 +
            min(request.ticketCount / 20, 1) * 0.2 +
            (1 - min(request.loginFrequency, 1)) * 0.1
        )
        
        risk_level = 'low' if risk_score < 0.3 else 'medium' if risk_score < 0.7 else 'high'
        
        return {
            'riskLevel': risk_level,
            'confidence': 0.75,
            'factors': [{'factor': 'Heuristic-based', 'importance': 1.0}]
        }
```

### Burnout Detection

```python
from river import anomaly, preprocessing
from datetime import datetime

class BurnoutDetectionRequest(BaseModel):
    userId: str
    tasksAssigned: int
    overtimeHours: float
    taskCompletionTime: float
    stressIndicators: int

burnout_detector = anomaly.HalfSpaceTrees()
scaler = preprocessing.StandardScaler()

@app.post('/api/ai/burnout-detection')
async def detect_burnout(request: BurnoutDetectionRequest):
    features = {
        'tasks': request.tasksAssigned,
        'overtime': request.overtimeHours,
        'completion_time': request.taskCompletionTime,
        'stress': request.stressIndicators
    }
    
    # Scale features
    scaled = scaler.learn_one(features).transform_one(features)
    
    # Get anomaly score
    score = burnout_detector.score_one(scaled)
    burnout_detector.learn_one(scaled)
    
    # Determine risk level
    if score > 0.7:
        risk = 'high'
        recommendations = [
            'Reduce task assignments',
            'Schedule time off',
            'Redistribute workload'
        ]
    elif score > 0.4:
        risk = 'medium'
        recommendations = [
            'Monitor workload closely',
            'Encourage breaks'
        ]
    else:
        risk = 'low'
        recommendations = ['Maintain current pace']
    
    return {
        'burnoutRisk': risk,
        'score': float(score),
        'recommendations': recommendations
    }
```

### Ticket Classification

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import re

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str

# Initialize vectorizer and classifier
vectorizer = TfidfVectorizer(max_features=1000)
classifier = MultinomialNB()

@app.post('/api/ai/classify-ticket')
async def classify_ticket(request: TicketClassificationRequest):
    text = f"{request.subject} {request.description}".lower()
    
    # Rule-based classification (can be replaced with trained model)
    categories = {
        'technical': ['error', 'bug', 'crash', 'not working', 'issue'],
        'account': ['password', 'login', 'access', 'permission'],
        'feature': ['request', 'enhancement', 'suggest', 'need'],
        'billing': ['payment', 'invoice', 'subscription', 'charge']
    }
    
    category_scores = {}
    for cat, keywords in categories.items():
        score = sum(1 for word in keywords if word in text)
        category_scores[cat] = score
    
    category = max(category_scores, key=category_scores.get)
    
    # Priority detection
    urgent_words = ['urgent', 'critical', 'asap', 'emergency', 'down']
    priority = 'high' if any(word in text for word in urgent_words) else 'medium'
    
    return {
        'category': category,
        'priority': priority,
        'confidence': 0.8,
        'suggestedAssignee': None  # Can be enhanced with team routing
    }
```

## Common Patterns

### Middleware for JWT Validation

```javascript
const jwt = require('jsonwebtoken');

const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }
  
  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) {
      return res.status(403).json({ error: 'Invalid token' });
    }
    req.user = user;
    next();
  });
};

const requireRole = (role) => {
  return (req, res, next) => {
    if (req.user.role !== role) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
};

module.exports = { authenticateToken, requireRole };
```

### Audit Logging

```javascript
const logActivity = async (userId, action, details) => {
  await axios.post(
    `${process.env.REACT_APP_API_URL}/api/audit`,
    {
      userId,
      action,
      details,
      timestamp: new Date().toISOString(),
      ipAddress: req.ip
    },
    { headers: { Authorization: `Bearer ${token}` } }
  );
};

// Usage
await logActivity(user.id, 'USER_DELETED', { deletedUserId: targetUserId });
```

## Troubleshooting

### JWT Token Expired
```javascript
// Implement token refresh
const refreshToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  const response = await axios.post(
    `${process.env.REACT_APP_API_URL}/api/auth/refresh`,
    { refreshToken }
  );
  localStorage.setItem('token', response.data.token);
  return response.data.token;
};

// Axios interceptor for auto-refresh
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      const newToken = await refreshToken();
      error.config.headers['Authorization'] = `Bearer ${newToken}`;
      return axios.request(error.config);
    }
    return Promise.reject(error);
  }
);
```

### MongoDB Connection Issues
```javascript
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000
    });
    console.log('MongoDB connected');
  } catch (err) {
    console.error('MongoDB connection error:', err);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Model Not Loading
```python
import os
import logging

logger = logging.getLogger(__name__)

def load_model_safe(model_path):
    try:
        model = joblib.load(model_path)
        logger.info(f'Model loaded from {model_path}')
        return model
    except FileNotFoundError:
        logger.warning(f'Model not found at {model_path}, using default')
        return None
    except Exception as e:
        logger.error(f'Error loading model: {e}')
        return None
```

### CORS Issues
```javascript
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

## Best Practices

1. **Always validate input data** on both frontend and backend
2. **Use environment variables** for all sensitive configuration
3. **Implement rate limiting** to prevent API abuse
4. **Log all admin actions** for audit trails
5. **Cache frequently accessed data** (user profiles, task lists)
6. **Batch ML predictions** when processing multiple users
7. **Use MongoDB indexes** on frequently queried fields (userId, taskId)
8. **Implement graceful degradation** if ML service is unavailable
