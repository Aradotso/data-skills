---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and automated task assignment
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with ML insights"
  - "integrate risk detection and anomaly detection"
  - "build task management with burnout analysis"
  - "deploy user management system with AI features"
  - "configure JWT authentication for user management"
  - "implement AI ticket classification system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines traditional user/task management with AI-powered insights. It provides:

- **User Management**: Role-based access control, authentication, and user profiles
- **Task Management**: Kanban boards, time tracking, and assignment workflows
- **Support Tickets**: AI-classified ticketing system with smart routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project insights
- **Admin Dashboard**: Comprehensive analytics and audit logging

The system uses a three-tier architecture:
- **Frontend**: React.js application
- **Backend**: Node.js REST API with MongoDB
- **ML Service**: FastAPI-based Python service with scikit-learn and River

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

Create `.env` file:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
# Production mode
npm run prod
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# Production
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Build for production
npm run build
```

## Key API Endpoints

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "email": "user@example.com",
  "password": "securePassword123",
  "name": "John Doe",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "securePassword123"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": { "id": "...", "email": "...", "role": "..." }
}
```

### User Management

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Get user by ID
GET /api/users/:id

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
  "title": "Implement feature X",
  "description": "Details...",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in_progress"
}

// Track time
POST /api/tasks/:id/time
{
  "duration": 3600, // seconds
  "description": "Worked on implementation"
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "subject": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// AI classify ticket
POST /api/ml/classify-ticket
{
  "subject": "Login issue",
  "description": "Cannot access dashboard"
}
// Returns: { "category": "technical", "priority": "high", "assignedDepartment": "IT" }
```

### AI Analytics Endpoints

```javascript
// Risk prediction
POST /api/ml/predict-risk
{
  "userId": "user_id",
  "recentActivity": [...]
}

// Anomaly detection
POST /api/ml/detect-anomaly
{
  "userId": "user_id",
  "behaviorData": {
    "loginTime": "02:30",
    "accessedSensitiveData": true,
    "unusualLocation": true
  }
}

// Burnout detection
GET /api/ml/burnout-analysis/:userId

// Project insights
POST /api/ml/project-insights
{
  "projectId": "project_id",
  "tasks": [...],
  "team": [...]
}
```

## Frontend Usage Patterns

### Authentication Hook

```javascript
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
      const res = await axios.get(`${API_URL}/auth/me`);
      setUser(res.data.user);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${API_URL}/auth/login`, { email, password });
    localStorage.setItem('token', res.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${res.data.token}`;
    setUser(res.data.user);
    return res.data;
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
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${API_URL}/tasks`);
      const grouped = {
        todo: res.data.filter(t => t.status === 'todo'),
        inProgress: res.data.filter(t => t.status === 'in_progress'),
        done: res.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/tasks/${taskId}/status`, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {Object.keys(tasks).map(column => (
        <div key={column} className="kanban-column">
          <h3>{column.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          {tasks[column].map(task => (
            <TaskCard 
              key={task.id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
      ))}
    </div>
  );
};
```

### AI Analytics Integration

```javascript
import axios from 'axios';

const ML_URL = process.env.REACT_APP_ML_URL;

export const useAIAnalytics = () => {
  const classifyTicket = async (subject, description) => {
    try {
      const res = await axios.post(`${ML_URL}/classify-ticket`, {
        subject,
        description
      });
      return res.data;
    } catch (error) {
      console.error('Ticket classification failed:', error);
      return null;
    }
  };

  const detectBurnout = async (userId) => {
    try {
      const res = await axios.get(`${ML_URL}/burnout-analysis/${userId}`);
      return res.data;
    } catch (error) {
      console.error('Burnout detection failed:', error);
      return null;
    }
  };

  const predictRisk = async (userId, activityData) => {
    try {
      const res = await axios.post(`${ML_URL}/predict-risk`, {
        userId,
        recentActivity: activityData
      });
      return res.data;
    } catch (error) {
      console.error('Risk prediction failed:', error);
      return null;
    }
  };

  return { classifyTicket, detectBurnout, predictRisk };
};
```

## Backend Implementation Patterns

### JWT Middleware

```javascript
const jwt = require('jsonwebtoken');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token provided' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = decoded.id;
    req.userRole = decoded.role;
    
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid authentication token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.userRole !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Task Controller

```javascript
const Task = require('../models/Task');
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL;

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      createdBy: req.userId,
      status: 'todo'
    });
    
    await task.save();
    
    // Get AI insights for task
    try {
      const insights = await axios.post(`${ML_SERVICE_URL}/task-insights`, {
        title,
        description,
        assignedTo
      });
      task.aiInsights = insights.data;
      await task.save();
    } catch (mlError) {
      console.error('ML service error:', mlError);
    }
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;
    
    const task = await Task.findById(id);
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    task.status = status;
    task.statusHistory.push({
      status,
      changedBy: req.userId,
      changedAt: new Date()
    });
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

## ML Service Implementation

### Ticket Classification

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib
import os

app = FastAPI()

MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketRequest(BaseModel):
    subject: str
    description: str

class TicketResponse(BaseModel):
    category: str
    priority: str
    assignedDepartment: str
    confidence: float

# Load or initialize models
try:
    vectorizer = joblib.load(f'{MODEL_PATH}/ticket_vectorizer.pkl')
    classifier = joblib.load(f'{MODEL_PATH}/ticket_classifier.pkl')
except:
    vectorizer = TfidfVectorizer(max_features=1000)
    classifier = MultinomialNB()

@app.post("/classify-ticket", response_model=TicketResponse)
async def classify_ticket(ticket: TicketRequest):
    try:
        text = f"{ticket.subject} {ticket.description}"
        features = vectorizer.transform([text])
        
        category = classifier.predict(features)[0]
        confidence = classifier.predict_proba(features).max()
        
        # Map category to department and priority
        department_map = {
            'technical': 'IT',
            'billing': 'Finance',
            'feature_request': 'Product',
            'general': 'Support'
        }
        
        priority = 'high' if confidence < 0.7 else 'medium'
        
        return TicketResponse(
            category=category,
            priority=priority,
            assignedDepartment=department_map.get(category, 'Support'),
            confidence=float(confidence)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Burnout Detection

```python
from pydantic import BaseModel
from typing import List
import numpy as np

class BurnoutRequest(BaseModel):
    userId: str
    weeklyHours: List[float]
    taskCompletionRate: float
    overtimeHours: float
    weekendWork: int

class BurnoutResponse(BaseModel):
    riskLevel: str
    score: float
    recommendations: List[str]

@app.post("/burnout-analysis/{user_id}", response_model=BurnoutResponse)
async def analyze_burnout(user_id: str):
    try:
        # Fetch user data from database
        user_data = await get_user_work_data(user_id)
        
        # Calculate burnout score
        avg_hours = np.mean(user_data['weeklyHours'])
        overtime_ratio = user_data['overtimeHours'] / (avg_hours * 4)
        
        score = (
            (avg_hours / 40) * 0.3 +
            overtime_ratio * 0.4 +
            (1 - user_data['taskCompletionRate']) * 0.2 +
            (user_data['weekendWork'] / 4) * 0.1
        )
        
        if score > 0.7:
            risk_level = 'high'
            recommendations = [
                'Reduce workload immediately',
                'Schedule time off',
                'Redistribute tasks to team members'
            ]
        elif score > 0.4:
            risk_level = 'medium'
            recommendations = [
                'Monitor workload closely',
                'Ensure regular breaks',
                'Consider task delegation'
            ]
        else:
            risk_level = 'low'
            recommendations = ['Maintain current work-life balance']
        
        return BurnoutResponse(
            riskLevel=risk_level,
            score=float(score),
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Anomaly Detection

```python
from river import anomaly
from datetime import datetime

class AnomalyRequest(BaseModel):
    userId: str
    loginTime: str
    accessedSensitiveData: bool
    unusualLocation: bool
    failedLoginAttempts: int

# Online learning anomaly detector
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyRequest):
    try:
        # Convert to feature vector
        hour = datetime.fromisoformat(request.loginTime).hour
        features = {
            'hour': hour,
            'sensitive_data': int(request.accessedSensitiveData),
            'unusual_location': int(request.unusualLocation),
            'failed_attempts': request.failedLoginAttempts
        }
        
        # Get anomaly score
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.6
        
        return {
            'isAnomaly': is_anomaly,
            'score': float(score),
            'actions': ['Flag for review', 'Notify admin'] if is_anomaly else []
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Configuration

### MongoDB Models

```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  name: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  isActive: { type: Boolean, default: true }
});

module.exports = mongoose.model('User', userSchema);

// models/Task.js
const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in_progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeTracked: { type: Number, default: 0 },
  statusHistory: [{
    status: String,
    changedBy: mongoose.Schema.Types.ObjectId,
    changedAt: Date
  }],
  aiInsights: mongoose.Schema.Types.Mixed
});

module.exports = mongoose.model('Task', taskSchema);
```

## Troubleshooting

### JWT Token Expired

```javascript
// Add token refresh logic
const refreshToken = async () => {
  try {
    const res = await axios.post(`${API_URL}/auth/refresh`, {
      refreshToken: localStorage.getItem('refreshToken')
    });
    localStorage.setItem('token', res.data.token);
    return res.data.token;
  } catch (error) {
    // Redirect to login
    window.location.href = '/login';
  }
};

// Axios interceptor for automatic token refresh
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

### ML Service Connection Issues

```javascript
// Add fallback when ML service is unavailable
const getMLPrediction = async (data) => {
  try {
    const res = await axios.post(`${ML_URL}/predict`, data, {
      timeout: 5000
    });
    return res.data;
  } catch (error) {
    console.warn('ML service unavailable, using fallback');
    return {
      prediction: 'medium',
      confidence: 0.5,
      source: 'fallback'
    };
  }
};
```

### MongoDB Connection Pooling

```javascript
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      maxPoolSize: 10,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Model Persistence for ML Service

```python
import joblib
import os
from pathlib import Path

MODEL_PATH = os.getenv('MODEL_PATH', './models')
Path(MODEL_PATH).mkdir(parents=True, exist_ok=True)

def save_model(model, name):
    """Save trained model to disk"""
    joblib.dump(model, f'{MODEL_PATH}/{name}.pkl')

def load_model(name):
    """Load model from disk or return None"""
    try:
        return joblib.load(f'{MODEL_PATH}/{name}.pkl')
    except:
        return None

# Auto-save models periodically
from fastapi_utils.tasks import repeat_every

@app.on_event("startup")
@repeat_every(seconds=3600)  # Every hour
async def save_models_task():
    save_model(vectorizer, 'ticket_vectorizer')
    save_model(classifier, 'ticket_classifier')
```

## Deployment

### Environment Variables Checklist

Backend `.env`:
- `PORT`
- `MONGODB_URI`
- `JWT_SECRET`
- `JWT_EXPIRE`
- `ML_SERVICE_URL`

ML Service `.env`:
- `MONGODB_URI`
- `MODEL_PATH`
- `LOG_LEVEL`

Frontend `.env`:
- `REACT_APP_API_URL`
- `REACT_APP_ML_URL`

### Production Build

```bash
# Build frontend
cd frontend
npm run build

# Serve with backend
cd ../backend
npm install express-static
# Add to server.js:
# app.use(express.static(path.join(__dirname, '../frontend/build')));
```
