---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management for enterprise organizations
triggers:
  - "help me build a user management system with AI analytics"
  - "how do I implement enterprise task management with AI insights"
  - "show me how to use the enterprise user management system"
  - "integrate AI-based risk detection in user management"
  - "create a kanban board with AI analytics"
  - "implement JWT authentication with role-based access control"
  - "set up AI ticket classification and routing"
  - "build an admin dashboard with ML predictions"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack JavaScript application combining React frontend, Node.js backend, and FastAPI ML service to provide intelligent user, task, and ticket management. It features AI-powered risk detection, anomaly analysis, burnout prediction, and automated ticket routing for enterprise environments.

**Key capabilities:**
- JWT-based authentication with role-based access control (RBAC)
- Kanban task management with time tracking
- Support ticket system with AI classification
- AI analytics: risk prediction, anomaly detection, burnout analysis
- MongoDB data persistence
- Real-time notifications and audit logging

## Installation

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
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
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
LOG_LEVEL=INFO
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
REACT_APP_ML_SERVICE_URL=http://localhost:8000
EOF

npm start
```

## Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   React     │─────▶│   Node.js   │─────▶│   MongoDB   │
│  Frontend   │      │   Backend   │      │  Database   │
└─────────────┘      └─────────────┘      └─────────────┘
       │                     │
       │                     ▼
       │             ┌─────────────┐
       └────────────▶│   FastAPI   │
                     │ ML Service  │
                     └─────────────┘
```

## Core API Endpoints

### Authentication

```javascript
// User Login
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "securepassword"
}

// Response
{
  "token": "jwt_token_here",
  "user": {
    "id": "user_id",
    "name": "John Doe",
    "role": "user"
  }
}

// User Registration
POST /api/auth/register
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "password": "password123",
  "role": "user"
}
```

### User Management (Admin)

```javascript
// Get All Users
GET /api/users
Headers: { Authorization: "Bearer ${JWT_TOKEN}" }

// Create User
POST /api/users
{
  "name": "New User",
  "email": "newuser@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update User
PUT /api/users/:userId
{
  "name": "Updated Name",
  "role": "admin"
}

// Delete User
DELETE /api/users/:userId
```

### Task Management

```javascript
// Get User Tasks
GET /api/tasks/user/:userId
Headers: { Authorization: "Bearer ${JWT_TOKEN}" }

// Create Task
POST /api/tasks
{
  "title": "Implement new feature",
  "description": "Add AI analytics dashboard",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update Task Status
PATCH /api/tasks/:taskId/status
{
  "status": "in-progress"
}

// Track Time
POST /api/tasks/:taskId/time
{
  "duration": 3600,  // seconds
  "date": "2026-04-15"
}
```

### Support Tickets

```javascript
// Create Ticket
POST /api/tickets
{
  "title": "System access issue",
  "description": "Cannot login to dashboard",
  "priority": "high",
  "category": "technical"
}

// Get User Tickets
GET /api/tickets/user/:userId

// Update Ticket
PATCH /api/tickets/:ticketId
{
  "status": "resolved",
  "resolution": "Password reset completed"
}
```

## AI/ML Integration

### Risk Prediction

```python
# ML Service endpoint
POST /api/ml/predict-risk
{
  "userId": "user_id",
  "features": {
    "loginFailures": 3,
    "unusualActivity": true,
    "dataAccessPatterns": [...]
  }
}

# Response
{
  "riskScore": 0.75,
  "riskLevel": "high",
  "factors": ["Multiple login failures", "Unusual access times"]
}
```

### Anomaly Detection

```python
POST /api/ml/detect-anomaly
{
  "userId": "user_id",
  "behavior": {
    "loginTime": "03:00 AM",
    "location": "Unknown",
    "dataAccessed": 500
  }
}

# Response
{
  "isAnomaly": true,
  "anomalyScore": 0.88,
  "type": "unusual_time_and_location"
}
```

### Burnout Detection

```python
POST /api/ml/detect-burnout
{
  "userId": "user_id",
  "metrics": {
    "workHours": 60,
    "taskLoad": 15,
    "completionRate": 0.4,
    "overtimeHours": 20
  }
}

# Response
{
  "burnoutRisk": "high",
  "score": 0.82,
  "recommendations": [
    "Reduce task load",
    "Schedule break time"
  ]
}
```

### Ticket Classification

```python
POST /api/ml/classify-ticket
{
  "title": "Cannot access payroll system",
  "description": "Getting error when trying to view pay stubs"
}

# Response
{
  "category": "technical",
  "priority": "medium",
  "suggestedAssignee": "IT Support",
  "confidence": 0.91
}
```

## Frontend Implementation

### Authentication Setup

```javascript
// src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const login = async (email, password) => {
  const response = await axios.post(`${API_URL}/api/auth/login`, {
    email,
    password
  });
  
  if (response.data.token) {
    localStorage.setItem('token', response.data.token);
    localStorage.setItem('user', JSON.stringify(response.data.user));
  }
  
  return response.data;
};

export const logout = () => {
  localStorage.removeItem('token');
  localStorage.removeItem('user');
};

export const getCurrentUser = () => {
  return JSON.parse(localStorage.getItem('user'));
};

export const getAuthHeader = () => {
  const token = localStorage.getItem('token');
  return token ? { Authorization: `Bearer ${token}` } : {};
};
```

### Protected Route Component

```javascript
// src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import { getCurrentUser } from '../services/authService';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const user = getCurrentUser();
  
  if (!user) {
    return <Navigate to="/login" />;
  }
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### Task Management Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${API_URL}/api/tasks/user/${userId}`,
        { headers: getAuthHeader() }
      );
      
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: getAuthHeader() }
      );
      
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="task-board">
      <div className="column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <TaskCard 
            key={task.id} 
            task={task} 
            onMove={moveTask}
          />
        ))}
      </div>
      
      <div className="column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <TaskCard 
            key={task.id} 
            task={task} 
            onMove={moveTask}
          />
        ))}
      </div>
      
      <div className="column">
        <h3>Done</h3>
        {tasks.done.map(task => (
          <TaskCard 
            key={task.id} 
            task={task} 
            onMove={moveTask}
          />
        ))}
      </div>
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { getAuthHeader } from '../services/authService';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  const ML_SERVICE_URL = process.env.REACT_APP_ML_SERVICE_URL;

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch risk prediction
      const riskResponse = await axios.post(
        `${ML_SERVICE_URL}/api/ml/predict-risk`,
        { userId },
        { headers: getAuthHeader() }
      );

      // Fetch burnout detection
      const burnoutResponse = await axios.post(
        `${ML_SERVICE_URL}/api/ml/detect-burnout`,
        { userId },
        { headers: getAuthHeader() }
      );

      setAnalytics({
        riskScore: riskResponse.data.riskScore,
        burnoutRisk: burnoutResponse.data.burnoutRisk,
        anomalies: riskResponse.data.factors || []
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const getRiskColor = (score) => {
    if (score < 0.3) return 'green';
    if (score < 0.7) return 'yellow';
    return 'red';
  };

  return (
    <div className="ai-analytics">
      <div className="metric-card">
        <h3>Risk Score</h3>
        <div 
          className="score"
          style={{ color: getRiskColor(analytics.riskScore) }}
        >
          {analytics.riskScore ? 
            `${(analytics.riskScore * 100).toFixed(0)}%` : 
            'Loading...'}
        </div>
      </div>

      <div className="metric-card">
        <h3>Burnout Risk</h3>
        <div className="risk-level">
          {analytics.burnoutRisk || 'Loading...'}
        </div>
      </div>

      <div className="anomalies-list">
        <h3>Detected Anomalies</h3>
        <ul>
          {analytics.anomalies.map((anomaly, index) => (
            <li key={index}>{anomaly}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Backend Implementation

### User Model (MongoDB)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  password: {
    type: String,
    required: true
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
  },
  lastLogin: Date,
  isActive: {
    type: Boolean,
    default: true
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      throw new Error('No token provided');
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user || !user.isActive) {
      throw new Error('User not found or inactive');
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
  }
};

const adminAuth = async (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { auth, adminAuth };
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      createdBy: req.user._id,
      status: 'todo'
    });

    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('createdBy', 'name email')
      .sort('-createdAt');
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.taskId);

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.status = status;
    task.updatedAt = Date.now();

    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const { duration, date } = req.body;
    const task = await Task.findById(req.params.taskId);

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.timeTracking.push({
      duration,
      date: date || Date.now(),
      user: req.user._id
    });

    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
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
    const { title, description, priority } = req.body;
    
    // Get AI classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/api/ml/classify-ticket`,
      { title, description }
    );

    const ticket = new Ticket({
      title,
      description,
      priority: priority || mlResponse.data.priority,
      category: mlResponse.data.category,
      createdBy: req.user._id,
      assignedTo: mlResponse.data.suggestedAssignee,
      status: 'open'
    });

    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.getUserTickets = async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.params.userId })
      .populate('assignedTo', 'name email')
      .sort('-createdAt');
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

## ML Service Implementation

### Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
import os

class RiskPredictor:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        self.model = self._load_or_create_model()
    
    def _load_or_create_model(self):
        if os.path.exists(self.model_path):
            return joblib.load(self.model_path)
        else:
            # Create new model
            model = RandomForestClassifier(n_estimators=100, random_state=42)
            return model
    
    def predict_risk(self, features):
        """
        Features: login_failures, unusual_activity, data_access_volume,
                 off_hours_access, failed_authentications
        """
        feature_vector = np.array([
            features.get('loginFailures', 0),
            1 if features.get('unusualActivity', False) else 0,
            features.get('dataAccessVolume', 0),
            features.get('offHoursAccess', 0),
            features.get('failedAuth', 0)
        ]).reshape(1, -1)
        
        # If model is trained
        if hasattr(self.model, 'classes_'):
            risk_proba = self.model.predict_proba(feature_vector)[0][1]
        else:
            # Simple heuristic if not trained
            risk_score = (
                features.get('loginFailures', 0) * 0.3 +
                (1 if features.get('unusualActivity', False) else 0) * 0.4 +
                min(features.get('failedAuth', 0) / 5, 1) * 0.3
            )
            risk_proba = min(risk_score, 1.0)
        
        risk_level = 'low' if risk_proba < 0.3 else 'medium' if risk_proba < 0.7 else 'high'
        
        factors = []
        if features.get('loginFailures', 0) > 2:
            factors.append('Multiple login failures')
        if features.get('unusualActivity', False):
            factors.append('Unusual activity pattern detected')
        if features.get('offHoursAccess', 0) > 5:
            factors.append('Excessive off-hours access')
        
        return {
            'riskScore': float(risk_proba),
            'riskLevel': risk_level,
            'factors': factors
        }
    
    def save_model(self):
        joblib.dump(self.model, self.model_path)
```

### Burnout Detection

```python
# ml-service/models/burnout_detector.py
import numpy as np

class BurnoutDetector:
    def __init__(self):
        self.thresholds = {
            'work_hours': 50,
            'overtime': 10,
            'task_load': 12,
            'completion_rate': 0.6
        }
    
    def detect_burnout(self, metrics):
        """
        Metrics: work_hours, overtime_hours, task_load, completion_rate
        """
        work_hours = metrics.get('workHours', 40)
        overtime = metrics.get('overtimeHours', 0)
        task_load = metrics.get('taskLoad', 5)
        completion_rate = metrics.get('completionRate', 1.0)
        
        # Calculate burnout score
        work_score = min(work_hours / self.thresholds['work_hours'], 1.5)
        overtime_score = min(overtime / self.thresholds['overtime'], 1.5)
        task_score = min(task_load / self.thresholds['task_load'], 1.5)
        completion_score = max(1 - completion_rate, 0) * 2
        
        burnout_score = (
            work_score * 0.3 +
            overtime_score * 0.25 +
            task_score * 0.25 +
            completion_score * 0.2
        )
        
        burnout_score = min(burnout_score, 1.0)
        
        if burnout_score < 0.4:
            risk_level = 'low'
        elif burnout_score < 0.7:
            risk_level = 'medium'
        else:
            risk_level = 'high'
        
        recommendations = []
        if work_hours > self.thresholds['work_hours']:
            recommendations.append('Reduce weekly work hours')
        if overtime > self.thresholds['overtime']:
            recommendations.append('Limit overtime work')
        if task_load > self.thresholds['task_load']:
            recommendations.append('Redistribute task load')
        if completion_rate < self.thresholds['completion_rate']:
            recommendations.append('Review task priorities and deadlines')
        
        return {
            'burnoutRisk': risk_level,
            'score': float(burnout_score),
            'recommendations': recommendations,
            'metrics': {
                'workHours': work_hours,
                'overtime': overtime,
                'taskLoad': task_load,
                'completionRate': completion_rate
            }
        }
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Dict, List, Optional
from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector
from models.ticket_classifier import TicketClassifier
import os

app = FastAPI(title="Enterprise ML Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize models
risk_predictor = RiskPredictor()
burnout_detector = BurnoutDetector()
ticket_classifier = TicketClassifier()

# Request models
class RiskPredictionRequest(BaseModel):
    userId: str
    features: Dict

class BurnoutDetectionRequest(BaseModel):
    userId: str
    metrics: Dict

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class AnomalyDetectionRequest(BaseModel):
    userId: str
    behavior: Dict

# Endpoints
@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        result = risk_predictor.predict_risk(request.features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        result = burnout_detector.detect_burnout(request.metrics)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        result = ticket_classifier.classify(
            request.title,
            request.description
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        behavior = request.behavior
        
        # Simple anomaly detection logic
        is_anomaly = False
        anomaly_score = 0.0
        anomaly_type = None
        
        # Check for unusual login time
        login_time = behavior.get('loginTime', '')
        if any(hour in login_time for hour in ['01:', '02:', '03:', '04:']):
            is_anomaly = True
            anomaly_score += 0.4
            anomaly_type = 'unusual_time'
        
        # Check for unknown location
        if behavior.get('location') == 'Unknown':
            is_anomaly = True
            anomaly_score += 0.3
            anomaly_type = 'unknown_location'
        
        # Check for excessive data access
        data_accessed = behavior.get('dataAccessed', 0)
        if data_accessed > 200:
            is_anomaly = True
            anomaly_score += 0.3
            anomaly_type = 'excessive_data_access'
        
        anomaly_score = min(anomaly_score, 1.0)
        
        return {
            'isAnomaly': is_anomaly,
            'anomalyScore': anomaly_score,
            'type': anomaly_type,
            'timestamp': behavior.get('timestamp')
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Common Patterns

### Role-Based Access Control

```javascript
// backend/middleware/roleCheck.js
const checkRole = (allowedRoles) => {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Authentication required' });
    }
    
    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ 
        error: 'Insufficient permissions' 
      });
    }
    
    next();
  };
};

// Usage in routes
router.post('/users', auth, checkRole(['admin']), userController.createUser);
router.get('/users', auth, checkRole(['admin', 'manager']), userController.getUsers);
```

### Audit Logging

```javascript
// backend/middleware/auditLog.js
const AuditLog = require('../models/AuditLog');

const auditLog = (action) => {
  return async (req, res, next) => {
    const originalJson = res.json;
    
    res.json = function(data) {
      // Log the action
      AuditLog.create({
        action,
        userId: req.user?._id,
        resource: req.originalUrl,
        method: req.method,
        ipAddress: req.ip,
        userAgent: req.get('user-agent'),
        timestamp: new Date(),
        success: res.statusCode < 400
      }).catch(err => console.error('Audit log error:', err));
      
      originalJson.call(this, data);
    };
    
    next();
  };
};

// Usage
router.delete('/users/:id', auth, adminAuth, auditLog('DELETE_USER'), userController.deleteUser);
```

### Real-time Notifications

```javascript
// backend/services/notificationService.js
const Notification = require('../models/Notification');

class NotificationService {
  
