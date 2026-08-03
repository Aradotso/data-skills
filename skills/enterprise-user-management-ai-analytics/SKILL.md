---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and automated insights for enterprise workflows
triggers:
  - "set up enterprise user management system"
  - "create user management dashboard with AI analytics"
  - "implement task tracking with burnout detection"
  - "build admin panel with role-based access control"
  - "add AI-powered ticket classification system"
  - "integrate anomaly detection for user behavior"
  - "configure user management with Kanban boards"
  - "deploy enterprise management system with ML service"
---

# Enterprise User Management AI Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with the Enterprise User Management System with AI Analytics, a full-stack application combining React frontend, Node.js backend, and FastAPI ML service to manage users, tasks, and support tickets with intelligent automation and predictive insights.

## What It Does

The Enterprise User Management System provides:
- **User Management**: Secure authentication, role-based access control, user CRUD operations
- **Task Management**: Kanban boards (To Do → In Progress → Done), time tracking, task assignment
- **Support Tickets**: Ticket creation, tracking, and AI-based classification/routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, predictive project insights
- **Admin Dashboard**: Organization analytics, audit logs, security alerts

## Installation

### Prerequisites
- Node.js (v14+)
- Python 3.8+
- MongoDB instance running

### Clone and Setup

```bash
# Clone repository
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
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
EOF

# Start backend server
npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file for ML service
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

ML service runs at `http://localhost:8000`

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

Frontend runs at `http://localhost:3000`

## Key API Endpoints

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "username": "john_doe",
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
// Response: { "token": "jwt_token_here", "user": {...} }
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer {token}" }

// Create user
POST /api/users
Headers: { "Authorization": "Bearer {token}" }
{
  "username": "jane_smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:id
Headers: { "Authorization": "Bearer {token}" }
{
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
Headers: { "Authorization": "Bearer {token}" }
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks/my-tasks
Headers: { "Authorization": "Bearer {token}" }

// Create task
POST /api/tasks
Headers: { "Authorization": "Bearer {token}" }
{
  "title": "Implement user authentication",
  "description": "Add JWT-based authentication",
  "assignedTo": "user_id_here",
  "status": "todo",
  "priority": "high",
  "dueDate": "2026-05-01"
}

// Update task status
PUT /api/tasks/:id/status
Headers: { "Authorization": "Bearer {token}" }
{
  "status": "in_progress"
}

// Log time on task
POST /api/tasks/:id/time-log
Headers: { "Authorization": "Bearer {token}" }
{
  "timeSpent": 7200 // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { "Authorization": "Bearer {token}" }
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when accessing admin panel",
  "priority": "medium",
  "category": "technical"
}

// Get tickets
GET /api/tickets
Headers: { "Authorization": "Bearer {token}" }

// Update ticket
PUT /api/tickets/:id
Headers: { "Authorization": "Bearer {token}" }
{
  "status": "in_progress",
  "assignedTo": "admin_user_id"
}
```

### AI/ML Endpoints

```python
# Classify ticket (FastAPI)
POST http://localhost:8000/classify-ticket
{
  "title": "Cannot login to system",
  "description": "Getting authentication error",
  "content": "User reports unable to access account after password reset"
}
# Response: { "category": "technical", "priority": "high", "confidence": 0.87 }

# Detect anomaly
POST http://localhost:8000/detect-anomaly
{
  "userId": "user_id_here",
  "activityLog": [
    {"action": "login", "timestamp": "2026-04-15T10:00:00Z", "ip": "192.168.1.1"},
    {"action": "data_access", "timestamp": "2026-04-15T10:05:00Z", "resource": "sensitive_data"}
  ]
}
# Response: { "isAnomaly": true, "riskScore": 0.73, "reason": "Unusual access pattern" }

# Predict burnout
POST http://localhost:8000/predict-burnout
{
  "userId": "user_id_here",
  "tasksCompleted": 45,
  "averageTaskTime": 14400,
  "overtimeHours": 25,
  "missedDeadlines": 3
}
# Response: { "burnoutRisk": "high", "score": 0.82, "recommendations": [...] }

# Project delay prediction
POST http://localhost:8000/predict-project-delay
{
  "projectId": "proj_123",
  "tasksTotal": 50,
  "tasksCompleted": 20,
  "daysRemaining": 15,
  "teamSize": 5
}
# Response: { "delayProbability": 0.65, "estimatedDelay": 5, "factors": [...] }
```

## Frontend Integration Examples

### React Authentication Hook

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
      const response = await axios.get(`${API_URL}/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/auth/login`, {
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

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    in_progress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/tasks/my-tasks`);
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], in_progress: [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`${API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const handleDrop = (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');
    
    if (currentStatus !== newStatus) {
      updateTaskStatus(taskId, newStatus);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  return (
    <div className="kanban-board">
      {['todo', 'in_progress', 'done'].map((status) => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          {tasks[status].map((task) => (
            <div
              key={task._id}
              className="task-card"
              draggable
              onDragStart={(e) => handleDragStart(e, task._id, status)}
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

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_SERVICE_URL = process.env.REACT_APP_ML_SERVICE_URL;
const API_URL = process.env.REACT_APP_API_URL;

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    burnoutRisk: null,
    anomalies: [],
    projectInsights: null
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch user metrics
      const userMetrics = await axios.get(`${API_URL}/users/${userId}/metrics`);
      
      // Check burnout risk
      const burnoutResponse = await axios.post(`${ML_SERVICE_URL}/predict-burnout`, {
        userId: userId,
        tasksCompleted: userMetrics.data.tasksCompleted,
        averageTaskTime: userMetrics.data.avgTaskTime,
        overtimeHours: userMetrics.data.overtimeHours,
        missedDeadlines: userMetrics.data.missedDeadlines
      });

      // Check for anomalies
      const anomalyResponse = await axios.post(`${ML_SERVICE_URL}/detect-anomaly`, {
        userId: userId,
        activityLog: userMetrics.data.recentActivity
      });

      setAnalytics({
        burnoutRisk: burnoutResponse.data,
        anomalies: anomalyResponse.data,
        projectInsights: userMetrics.data.projectStats
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <h2>AI Insights</h2>
      
      {analytics.burnoutRisk && (
        <div className={`burnout-alert ${analytics.burnoutRisk.burnoutRisk}`}>
          <h3>Burnout Risk: {analytics.burnoutRisk.burnoutRisk}</h3>
          <p>Score: {(analytics.burnoutRisk.score * 100).toFixed(0)}%</p>
          <ul>
            {analytics.burnoutRisk.recommendations?.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      )}

      {analytics.anomalies?.isAnomaly && (
        <div className="anomaly-warning">
          <h3>⚠️ Anomaly Detected</h3>
          <p>Risk Score: {(analytics.anomalies.riskScore * 100).toFixed(0)}%</p>
          <p>{analytics.anomalies.reason}</p>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Backend Implementation Examples

### User Model (MongoDB Schema)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const UserSchema = new mongoose.Schema({
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
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) {
    next();
  }
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

// Generate JWT token
UserSchema.methods.getSignedJwtToken = function() {
  return jwt.sign(
    { id: this._id, role: this.role },
    process.env.JWT_SECRET,
    { expiresIn: process.env.JWT_EXPIRE }
  );
};

// Match password
UserSchema.methods.matchPassword = async function(enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const User = require('../models/User');

// Get user's tasks
exports.getMyTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ 
      assignedTo: req.user.id 
    }).populate('assignedBy', 'username email');
    
    res.status(200).json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

// Create task
exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = await Task.create({
      title,
      description,
      assignedTo,
      assignedBy: req.user.id,
      priority,
      dueDate,
      status: 'todo'
    });
    
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: 'Invalid task data', error: error.message });
  }
};

// Update task status
exports.updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;
    
    const task = await Task.findById(id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    // Check authorization
    if (task.assignedTo.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Not authorized' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    
    res.status(200).json(task);
  } catch (error) {
    res.status(400).json({ message: 'Update failed', error: error.message });
  }
};

// Log time on task
exports.logTime = async (req, res) => {
  try {
    const { id } = req.params;
    const { timeSpent } = req.body; // in seconds
    
    const task = await Task.findById(id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.timeSpent = (task.timeSpent || 0) + timeSpent;
    task.timeLogs = task.timeLogs || [];
    task.timeLogs.push({
      userId: req.user.id,
      duration: timeSpent,
      loggedAt: new Date()
    });
    
    await task.save();
    
    res.status(200).json(task);
  } catch (error) {
    res.status(400).json({ message: 'Time log failed', error: error.message });
  }
};
```

### Authentication Middleware

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
    
    if (!req.user) {
      return res.status(401).json({ message: 'User not found' });
    }
    
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Token is invalid or expired' });
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

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly
import pickle
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Enable CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models directory
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

# Pydantic models
class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    content: str

class AnomalyDetectionRequest(BaseModel):
    userId: str
    activityLog: List[Dict]

class BurnoutPredictionRequest(BaseModel):
    userId: str
    tasksCompleted: int
    averageTaskTime: float
    overtimeHours: float
    missedDeadlines: int

class ProjectDelayRequest(BaseModel):
    projectId: str
    tasksTotal: int
    tasksCompleted: int
    daysRemaining: int
    teamSize: int

# Initialize anomaly detector
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)

@app.get("/")
def read_root():
    return {"message": "Enterprise User Management ML Service", "status": "running"}

@app.post("/classify-ticket")
def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket by category and priority"""
    try:
        # Simple rule-based classification (replace with trained model)
        text = f"{request.title} {request.description} {request.content}".lower()
        
        # Category classification
        if any(word in text for word in ['login', 'password', 'access', 'authentication']):
            category = 'technical'
        elif any(word in text for word in ['permission', 'role', 'access denied']):
            category = 'security'
        elif any(word in text for word in ['feature', 'request', 'enhancement']):
            category = 'feature_request'
        else:
            category = 'general'
        
        # Priority classification
        if any(word in text for word in ['urgent', 'critical', 'down', 'broken']):
            priority = 'high'
        elif any(word in text for word in ['important', 'soon', 'asap']):
            priority = 'medium'
        else:
            priority = 'low'
        
        confidence = 0.85  # Mock confidence score
        
        return {
            "category": category,
            "priority": priority,
            "confidence": confidence
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    try:
        # Extract features from activity log
        if not request.activityLog:
            return {"isAnomaly": False, "riskScore": 0.0, "reason": "No activity"}
        
        # Simple feature extraction
        features = {
            'login_count': sum(1 for log in request.activityLog if log.get('action') == 'login'),
            'data_access_count': sum(1 for log in request.activityLog if log.get('action') == 'data_access'),
            'failed_attempts': sum(1 for log in request.activityLog if log.get('status') == 'failed')
        }
        
        # Calculate risk score
        risk_score = 0.0
        reasons = []
        
        if features['failed_attempts'] > 5:
            risk_score += 0.4
            reasons.append("High number of failed login attempts")
        
        if features['login_count'] > 20:
            risk_score += 0.3
            reasons.append("Unusual login frequency")
        
        if features['data_access_count'] > 50:
            risk_score += 0.3
            reasons.append("Excessive data access")
        
        is_anomaly = risk_score > 0.5
        
        return {
            "isAnomaly": is_anomaly,
            "riskScore": min(risk_score, 1.0),
            "reason": "; ".join(reasons) if reasons else "Normal activity"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-burnout")
def predict_burnout(request: BurnoutPredictionRequest):
    """Predict employee burnout risk"""
    try:
        # Calculate burnout score based on metrics
        score = 0.0
        recommendations = []
        
        # Task completion rate
        if request.tasksCompleted > 40:
            score += 0.2
            recommendations.append("High task completion rate - consider workload distribution")
        
        # Average task time (in seconds)
        if request.averageTaskTime > 28800:  # 8 hours
            score += 0.3
            recommendations.append("Tasks taking longer than expected - may need support")
        
        # Overtime hours
        if request.overtimeHours > 20:
            score += 0.3
            recommendations.append("Excessive overtime detected - review work-life balance")
        
        # Missed deadlines
        if request.missedDeadlines > 2:
            score += 0.2
            recommendations.append("Multiple missed deadlines - may indicate overload")
        
        # Determine risk level
        if score >= 0.7:
            risk = "high"
        elif score >= 0.4:
            risk = "medium"
        else:
            risk = "low"
        
        if not recommendations:
            recommendations.append("Workload appears balanced")
        
        return {
            "burnoutRisk": risk,
            "score": min(score, 1.0),
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-project-delay")
def predict_project_delay(request: ProjectDelayRequest):
    """Predict likelihood of project delay"""
    try:
        # Calculate completion rate
        completion_rate = request.tasksCompleted / request.tasksTotal if request.tasksTotal > 0 else 0
        
        # Expected completion rate
        expected_rate = 1.0 - (request.daysRemaining / 30.0)  # Assuming 30-day project
        
        # Calculate delay probability
        delay_prob = 0.0
        factors = []
        
        if completion_rate < expected_rate - 0.2:
            delay_prob += 0.5
            factors.append("Behind schedule")
        
        if request.teamSize < 3:
            delay_prob += 0.15
            factors.append("Small team size")
        
        if request.daysRemaining < 7 and completion_rate < 0.8:
            delay_prob += 0.35
            factors.append("Tight deadline with low completion")
        
        # Estimate delay in days
        if delay_prob > 0.5:
            estimated_delay = int((1 - completion_rate) * request.daysRemaining * 0.5)
        else:
            estimated_delay = 0
        
        if not factors:
            factors.append("On track")
        
        return {
            "delayProbability": min(delay_prob, 1.0),
            "estimatedDelay": estimated_delay,
            "factors": factors
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Configuration

### Backend Environment Variables

```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_secure_random_string_here
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
```

### Frontend Environment Variables

```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

### ML Service Environment Variables

```bash
# ml-service/.env
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Common Patterns

### Protected Route Component

```javascript
// src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, allowedRoles }) => {
  const { user, loading } = useAuth();

  if (loading) {
    return <div>Loading...</div>;
  }

  if (!user) {
    return <Navigate to="/login" />;
  }

  if (allowedRoles && !allowedRoles.includes(user.role)) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;
```

### API Service Layer

```javascript
// src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to requests
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise
