---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create task tracking with kanban board"
  - "build support ticket system with AI classification"
  - "add burnout detection and risk prediction"
  - "configure JWT authentication for user roles"
  - "integrate ML service with user management backend"
  - "deploy enterprise user management with AI features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack JavaScript application that combines enterprise user management with AI-powered analytics. The system provides role-based access control, Kanban task tracking, support ticket management, and ML-driven insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

## What This Project Does

This system provides three main components:

1. **Frontend (React)**: User and admin dashboards with Kanban boards, time tracking, and ticket management
2. **Backend (Node.js)**: REST APIs for user management, authentication (JWT), task management, and ticket routing
3. **ML Service (FastAPI)**: AI models for ticket classification, risk prediction, anomaly detection, and burnout analysis

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+ and pip
- MongoDB instance running

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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=24h
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MODEL_PATH=./models
REDIS_URL=redis://localhost:6379
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
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
# Frontend runs at http://localhost:3000
```

## Key API Endpoints

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@company.com",
  "password": "securepass",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@company.com",
  "password": "securepass"
}
// Returns: { token, user }
```

### User Management (Admin only)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer <token>" }

// Update user
PUT /api/users/:id
{
  "name": "John Updated",
  "role": "admin",
  "status": "active"
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
  "description": "Complete by Friday",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo", // todo, in-progress, done
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
}

// Get user tasks
GET /api/tasks/user/:userId
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// AI classification happens automatically
// Returns ticket with AI-assigned category and priority
```

### AI Analytics

```javascript
// Risk prediction
POST /api/ai/risk-prediction
{
  "userId": "user_id",
  "features": {
    "failedLoginAttempts": 3,
    "unusualAccessHours": 5,
    "dataAccessVolume": 1500
  }
}
// Returns: { riskScore, riskLevel, recommendations }

// Burnout detection
POST /api/ai/burnout-detection
{
  "userId": "user_id",
  "workloadData": {
    "tasksCompleted": 45,
    "hoursWorked": 65,
    "weekendWork": 12,
    "missedDeadlines": 3
  }
}
// Returns: { burnoutScore, status, suggestions }

// Anomaly detection
POST /api/ai/anomaly-detection
{
  "userId": "user_id",
  "activityLog": [
    { "timestamp": "2026-04-15T02:30:00Z", "action": "data_export" },
    { "timestamp": "2026-04-15T02:35:00Z", "action": "bulk_delete" }
  ]
}
// Returns: { isAnomaly, anomalyScore, flaggedActivities }
```

## Frontend Usage Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(res.data);
    } catch (error) {
      logout();
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    setToken(res.data.token);
    setUser(res.data.user);
    localStorage.setItem('token', res.data.token);
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, isAdmin: user?.role === 'admin' }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const res = await axios.get(`${process.env.REACT_APP_API_URL}/tasks/user/${userId}`);
    const grouped = res.data.reduce((acc, task) => {
      acc[task.status].push(task);
      return acc;
    }, { todo: [], 'in-progress': [], done: [] });
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await axios.patch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
      status: newStatus
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card" draggable
              onDragStart={(e) => e.dataTransfer.setData('taskId', task._id)}
              onDragOver={(e) => e.preventDefault()}
              onDrop={(e) => {
                const taskId = e.dataTransfer.getData('taskId');
                moveTask(taskId, status);
              }}>
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-${task.priority}`}>{task.priority}</span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      const [risk, burnout, anomalies] = await Promise.all([
        axios.post(`${process.env.REACT_APP_API_URL}/ai/risk-prediction`, { userId }),
        axios.post(`${process.env.REACT_APP_API_URL}/ai/burnout-detection`, { userId }),
        axios.post(`${process.env.REACT_APP_API_URL}/ai/anomaly-detection`, { userId })
      ]);

      setAnalytics({
        risk: risk.data,
        burnout: burnout.data,
        anomalies: anomalies.data
      });
    } catch (error) {
      console.error('Analytics fetch error:', error);
    }
  };

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="metric-card">
        <h3>Risk Level</h3>
        <div className={`risk-${analytics.risk.riskLevel}`}>
          {analytics.risk.riskLevel.toUpperCase()}
        </div>
        <p>Score: {analytics.risk.riskScore}</p>
      </div>

      <div className="metric-card">
        <h3>Burnout Status</h3>
        <div className={`burnout-${analytics.burnout.status}`}>
          {analytics.burnout.status}
        </div>
        <ul>
          {analytics.burnout.suggestions.map((s, i) => (
            <li key={i}>{s}</li>
          ))}
        </ul>
      </div>

      <div className="metric-card">
        <h3>Anomaly Detection</h3>
        {analytics.anomalies.isAnomaly && (
          <div className="alert">
            <p>Unusual activity detected!</p>
            <p>Score: {analytics.anomalies.anomalyScore}</p>
          </div>
        )}
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Backend Patterns

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
    return res.status(401).json({ message: 'Not authorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Token invalid' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: 'Access denied' });
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

    // Trigger AI prediction for project delay
    if (task.dueDate) {
      const prediction = await axios.post(`${process.env.ML_SERVICE_URL}/predict-delay`, {
        taskData: task
      });
      task.delayPrediction = prediction.data;
      await task.save();
    }

    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('createdBy', 'name email')
      .sort('-createdAt');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

## ML Service Implementation

### Ticket Classification

```python
# ml-service/models/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import pickle
import os

class TicketClassifier:
    def __init__(self):
        self.vectorizer = TfidfVectorizer(max_features=1000)
        self.classifier = MultinomialNB()
        self.categories = ['technical', 'billing', 'account', 'general']
        
    def train(self, texts, labels):
        X = self.vectorizer.fit_transform(texts)
        self.classifier.fit(X, labels)
        
    def predict(self, text):
        X = self.vectorizer.transform([text])
        category = self.classifier.predict(X)[0]
        probabilities = self.classifier.predict_proba(X)[0]
        
        return {
            'category': category,
            'confidence': float(max(probabilities)),
            'all_probabilities': dict(zip(self.categories, probabilities))
        }
    
    def save(self, path):
        os.makedirs(path, exist_ok=True)
        with open(f'{path}/vectorizer.pkl', 'wb') as f:
            pickle.dump(self.vectorizer, f)
        with open(f'{path}/classifier.pkl', 'wb') as f:
            pickle.dump(self.classifier, f)
```

### FastAPI Main Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Dict
import numpy as np
from models.ticket_classifier import TicketClassifier
from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector

app = FastAPI(title="Enterprise User Management ML Service")

# Initialize models
ticket_classifier = TicketClassifier()
risk_predictor = RiskPredictor()
burnout_detector = BurnoutDetector()

class TicketRequest(BaseModel):
    title: str
    description: str

class RiskRequest(BaseModel):
    userId: str
    features: Dict[str, float]

class BurnoutRequest(BaseModel):
    userId: str
    workloadData: Dict[str, int]

@app.post("/classify-ticket")
async def classify_ticket(request: TicketRequest):
    text = f"{request.title} {request.description}"
    result = ticket_classifier.predict(text)
    
    # Auto-assign priority based on keywords
    high_priority_keywords = ['urgent', 'critical', 'down', 'error', 'cannot access']
    priority = 'high' if any(kw in text.lower() for kw in high_priority_keywords) else 'medium'
    
    return {
        **result,
        'suggestedPriority': priority
    }

@app.post("/risk-prediction")
async def predict_risk(request: RiskRequest):
    features = np.array([
        request.features.get('failedLoginAttempts', 0),
        request.features.get('unusualAccessHours', 0),
        request.features.get('dataAccessVolume', 0),
        request.features.get('privilegeEscalations', 0)
    ]).reshape(1, -1)
    
    risk_score = risk_predictor.predict(features)
    
    if risk_score > 0.8:
        risk_level = 'critical'
        recommendations = ['Immediate security review required', 'Restrict access temporarily']
    elif risk_score > 0.5:
        risk_level = 'high'
        recommendations = ['Monitor user activity closely', 'Review access logs']
    else:
        risk_level = 'low'
        recommendations = ['Continue normal monitoring']
    
    return {
        'riskScore': float(risk_score),
        'riskLevel': risk_level,
        'recommendations': recommendations
    }

@app.post("/burnout-detection")
async def detect_burnout(request: BurnoutRequest):
    workload = request.workloadData
    
    # Calculate burnout score (0-1)
    factors = {
        'overwork': min(workload.get('hoursWorked', 40) / 80, 1),
        'weekend_work': min(workload.get('weekendWork', 0) / 20, 1),
        'missed_deadlines': min(workload.get('missedDeadlines', 0) / 10, 1),
        'task_load': min(workload.get('tasksCompleted', 0) / 100, 1)
    }
    
    burnout_score = sum(factors.values()) / len(factors)
    
    if burnout_score > 0.7:
        status = 'high_risk'
        suggestions = [
            'Reduce task assignments',
            'Schedule mandatory break',
            'Redistribute workload'
        ]
    elif burnout_score > 0.4:
        status = 'moderate'
        suggestions = ['Monitor workload', 'Ensure work-life balance']
    else:
        status = 'healthy'
        suggestions = ['Maintain current pace']
    
    return {
        'burnoutScore': float(burnout_score),
        'status': status,
        'suggestions': suggestions,
        'factors': factors
    }

@app.post("/anomaly-detection")
async def detect_anomaly(request: dict):
    activity_log = request.get('activityLog', [])
    
    # Simple anomaly detection based on patterns
    flagged = []
    anomaly_score = 0
    
    for activity in activity_log:
        hour = int(activity['timestamp'].split('T')[1].split(':')[0])
        
        # Flag unusual hours (2 AM - 5 AM)
        if 2 <= hour <= 5:
            flagged.append({**activity, 'reason': 'Unusual access time'})
            anomaly_score += 0.3
        
        # Flag sensitive actions
        sensitive_actions = ['data_export', 'bulk_delete', 'privilege_change']
        if activity['action'] in sensitive_actions:
            flagged.append({**activity, 'reason': 'Sensitive action'})
            anomaly_score += 0.4
    
    return {
        'isAnomaly': anomaly_score > 0.5,
        'anomalyScore': min(anomaly_score, 1.0),
        'flaggedActivities': flagged
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Configuration

### MongoDB Models

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const UserSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true, select: false },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now }
});

UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
});

UserSchema.methods.matchPassword = async function(password) {
  return await bcrypt.compare(password, this.password);
};

UserSchema.methods.getSignedJwtToken = function() {
  return jwt.sign({ id: this._id }, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_EXPIRE
  });
};

module.exports = mongoose.model('User', UserSchema);
```

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
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
  dueDate: Date,
  timeTracked: { type: Number, default: 0 }, // in minutes
  delayPrediction: Object,
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date
});

module.exports = mongoose.model('Task', TaskSchema);
```

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const TicketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'] 
  },
  category: { 
    type: String, 
    enum: ['technical', 'billing', 'account', 'general'] 
  },
  aiClassification: Object,
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', TicketSchema);
```

## Common Patterns

### Protected Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');
const taskController = require('../controllers/taskController');

router.post('/', protect, taskController.createTask);
router.get('/user/:userId', protect, taskController.getUserTasks);
router.patch('/:id/status', protect, taskController.updateTaskStatus);
router.delete('/:id', protect, authorize('admin'), taskController.deleteTask);

module.exports = router;
```

### Error Handling

```javascript
// backend/middleware/errorHandler.js
const errorHandler = (err, req, res, next) => {
  let error = { ...err };
  error.message = err.message;

  // Mongoose bad ObjectId
  if (err.name === 'CastError') {
    error.message = 'Resource not found';
    error.statusCode = 404;
  }

  // Mongoose duplicate key
  if (err.code === 11000) {
    error.message = 'Duplicate field value entered';
    error.statusCode = 400;
  }

  // Mongoose validation error
  if (err.name === 'ValidationError') {
    error.message = Object.values(err.errors).map(e => e.message).join(', ');
    error.statusCode = 400;
  }

  res.status(error.statusCode || 500).json({
    success: false,
    message: error.message || 'Server Error'
  });
};

module.exports = errorHandler;
```

## Troubleshooting

### JWT Token Expires Too Quickly

Check `JWT_EXPIRE` in backend `.env`:

```env
JWT_EXPIRE=7d  # Instead of 24h for longer sessions
```

### ML Service Connection Refused

Ensure ML service is running and backend has correct URL:

```env
# backend/.env
ML_SERVICE_URL=http://localhost:8000
```

Verify ML service is accessible:

```bash
curl http://localhost:8000/health
```

### CORS Errors in Frontend

Add CORS middleware in backend:

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### MongoDB Connection Issues

Check MongoDB URI format:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
# OR for MongoDB Atlas:
MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/dbname
```

### Tasks Not Updating in Kanban Board

Ensure WebSocket or polling is implemented for real-time updates:

```javascript
// Frontend polling pattern
useEffect(() => {
  const interval = setInterval(() => {
    fetchTasks();
  }, 5000); // Poll every 5 seconds
  
  return () => clearInterval(interval);
}, []);
```

### AI Predictions Not Accurate

Retrain models with more data:

```python
# ml-service/train_models.py
from models.ticket_classifier import TicketClassifier
import pandas as pd

# Load training data
df = pd.read_csv('training_data/tickets.csv')

classifier = TicketClassifier()
classifier.train(df['text'].values, df['category'].values)
classifier.save('./models')
```

### High Memory Usage in ML Service

Implement model caching and lazy loading:

```python
# ml-service/main.py
from functools import lru_cache

@lru_cache(maxsize=1)
def get_ticket_classifier():
    classifier = TicketClassifier()
    classifier.load('./models')
    return classifier

@app.post("/classify-ticket")
async def classify_ticket(request: TicketRequest):
    classifier = get_ticket_classifier()
    return classifier.predict(request.text)
```
