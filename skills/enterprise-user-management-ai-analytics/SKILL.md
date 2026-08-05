---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "create admin dashboard with AI insights"
  - "implement role-based access control with task management"
  - "build user management app with ML predictions"
  - "add AI ticket classification to support system"
  - "configure burnout detection for enterprise users"
  - "setup Kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a full-stack enterprise user management platform with integrated AI/ML capabilities for intelligent task assignment, risk detection, anomaly analysis, and burnout prediction. Built with React frontend, Node.js backend, and FastAPI ML service using MongoDB.

## What It Does

- **User Management**: CRUD operations for users with role-based access control (Admin/User)
- **Task Management**: Kanban-style task boards with time tracking and assignment
- **Support Tickets**: Ticketing system with AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay forecasting
- **Authentication**: JWT-based secure authentication system
- **Audit Logging**: Track all system activities for compliance

## Installation

### Prerequisites

Ensure you have Node.js (v14+), Python (3.8+), and MongoDB installed.

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
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MODEL_PATH=./models
DB_URL=mongodb://localhost:27017/enterprise-ums
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Key API Endpoints

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "username": "john.doe",
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
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer <JWT_TOKEN>" }

// Create user
POST /api/users
{
  "username": "jane.smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
{
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:userId
```

### Task Management

```javascript
// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Build new dashboard component",
  "assignedTo": "userId123",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:taskId/status
{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:taskId/time
{
  "timeSpent": 7200 // seconds
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

// AI classify ticket
POST /api/ml/classify-ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when logging in"
}
// Response: { "category": "technical", "priority": "high", "assignTo": "IT Support" }
```

### AI Analytics Endpoints

```javascript
// Risk prediction
POST /api/ml/predict-risk
{
  "userId": "user123",
  "loginAttempts": 5,
  "dataAccessPattern": "unusual",
  "offHoursActivity": true
}

// Anomaly detection
POST /api/ml/detect-anomaly
{
  "userId": "user123",
  "activityLog": [
    { "action": "login", "timestamp": "2026-04-15T02:30:00Z" },
    { "action": "bulk_download", "timestamp": "2026-04-15T02:35:00Z" }
  ]
}

// Burnout analysis
POST /api/ml/analyze-burnout
{
  "userId": "user123",
  "tasksCompleted": 45,
  "avgWorkHours": 11.5,
  "weekendWork": 6,
  "overtimeDays": 12
}

// Project delay prediction
POST /api/ml/predict-delay
{
  "projectId": "proj123",
  "tasksRemaining": 15,
  "daysUntilDeadline": 10,
  "teamSize": 5,
  "completionRate": 0.6
}
```

## Code Examples

### Frontend: User Authentication Hook

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
      fetchCurrentUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchCurrentUser = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    setUser(user);
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return { user, loading, login, logout };
};
```

### Frontend: Task Kanban Board Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const TaskBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks`);
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], 'in-progress': [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/api/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const onDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const onDrop = (e, newStatus) => {
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');
    if (currentStatus !== newStatus) {
      updateTaskStatus(taskId, newStatus);
    }
  };

  const onDragOver = (e) => {
    e.preventDefault();
  };

  return (
    <div className="task-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div
          key={status}
          className="task-column"
          onDrop={(e) => onDrop(e, status)}
          onDragOver={onDragOver}
        >
          <h3>{status.toUpperCase().replace('-', ' ')}</h3>
          {tasks[status].map(task => (
            <div
              key={task._id}
              draggable
              onDragStart={(e) => onDragStart(e, task._id, status)}
              className="task-card"
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority ${task.priority}`}>
                {task.priority}
              </span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default TaskBoard;
```

### Backend: User Routes with JWT Auth

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { authenticateToken, isAdmin } = require('../middleware/auth');

// Get all users (Admin only)
router.get('/', authenticateToken, isAdmin, async (req, res) => {
  try {
    const users = await User.find()
      .select('-password')
      .populate('assignedTasks', 'title status');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create user (Admin only)
router.post('/', authenticateToken, isAdmin, async (req, res) => {
  try {
    const { username, email, password, role, department } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const user = new User({
      username,
      email,
      password,
      role: role || 'user',
      department
    });

    await user.save();
    
    // Log audit trail
    await logAudit({
      action: 'USER_CREATED',
      performedBy: req.user.id,
      targetUser: user._id,
      timestamp: new Date()
    });

    res.status(201).json({
      id: user._id,
      username: user.username,
      email: user.email,
      role: user.role
    });
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Update user
router.put('/:id', authenticateToken, isAdmin, async (req, res) => {
  try {
    const { role, status, department } = req.body;
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { role, status, department },
      { new: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json(user);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Delete user
router.delete('/:id', authenticateToken, isAdmin, async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Backend: Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authenticateToken = async (req, res, next) => {
  try {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1];

    if (!token) {
      return res.status(401).json({ message: 'Access token required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');

    if (!user) {
      return res.status(401).json({ message: 'User not found' });
    }

    req.user = user;
    next();
  } catch (error) {
    return res.status(403).json({ message: 'Invalid token' });
  }
};

const isAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authenticateToken, isAdmin };
```

### ML Service: Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from river import compose, linear_model, preprocessing
import joblib
import os

class RiskPredictor:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        if os.path.exists(model_path):
            self.model = joblib.load(model_path)
        else:
            self.model = compose.Pipeline(
                preprocessing.StandardScaler(),
                linear_model.LogisticRegression()
            )
    
    def predict(self, features):
        """
        Predict risk level based on user behavior
        Features: login_attempts, off_hours_activity, failed_auth, data_access_volume
        """
        risk_score = self.model.predict_proba_one(features)
        return {
            'risk_level': 'high' if risk_score.get(1, 0) > 0.7 else 'medium' if risk_score.get(1, 0) > 0.4 else 'low',
            'probability': risk_score.get(1, 0),
            'factors': self._identify_factors(features)
        }
    
    def train(self, features, label):
        """Online learning - update model with new data"""
        self.model.learn_one(features, label)
        joblib.dump(self.model, self.model_path)
    
    def _identify_factors(self, features):
        factors = []
        if features.get('login_attempts', 0) > 5:
            factors.append('Excessive login attempts')
        if features.get('off_hours_activity', 0) > 0:
            factors.append('Off-hours activity detected')
        if features.get('failed_auth', 0) > 3:
            factors.append('Multiple failed authentications')
        if features.get('data_access_volume', 0) > 1000:
            factors.append('Unusual data access volume')
        return factors
```

### ML Service: FastAPI Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.risk_predictor import RiskPredictor
from models.burnout_analyzer import BurnoutAnalyzer
from models.ticket_classifier import TicketClassifier
import uvicorn

app = FastAPI(title="Enterprise UMS ML Service")

risk_predictor = RiskPredictor()
burnout_analyzer = BurnoutAnalyzer()
ticket_classifier = TicketClassifier()

class RiskInput(BaseModel):
    userId: str
    loginAttempts: int
    offHoursActivity: int
    failedAuth: int
    dataAccessVolume: int

class BurnoutInput(BaseModel):
    userId: str
    tasksCompleted: int
    avgWorkHours: float
    weekendWork: int
    overtimeDays: int

class TicketInput(BaseModel):
    title: str
    description: str

@app.post("/api/ml/predict-risk")
async def predict_risk(data: RiskInput):
    try:
        features = {
            'login_attempts': data.loginAttempts,
            'off_hours_activity': data.offHoursActivity,
            'failed_auth': data.failedAuth,
            'data_access_volume': data.dataAccessVolume
        }
        prediction = risk_predictor.predict(features)
        return {
            'userId': data.userId,
            'riskLevel': prediction['risk_level'],
            'probability': prediction['probability'],
            'factors': prediction['factors']
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/analyze-burnout")
async def analyze_burnout(data: BurnoutInput):
    try:
        features = {
            'tasks_completed': data.tasksCompleted,
            'avg_work_hours': data.avgWorkHours,
            'weekend_work': data.weekendWork,
            'overtime_days': data.overtimeDays
        }
        analysis = burnout_analyzer.analyze(features)
        return {
            'userId': data.userId,
            'burnoutRisk': analysis['risk_level'],
            'score': analysis['score'],
            'recommendations': analysis['recommendations']
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(data: TicketInput):
    try:
        classification = ticket_classifier.classify(data.title, data.description)
        return {
            'category': classification['category'],
            'priority': classification['priority'],
            'assignTo': classification['assignTo'],
            'confidence': classification['confidence']
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Service"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### ML Service: Burnout Analyzer

```python
# ml-service/models/burnout_analyzer.py
import numpy as np

class BurnoutAnalyzer:
    def __init__(self):
        self.thresholds = {
            'max_work_hours': 9.0,
            'max_weekend_work': 2,
            'max_overtime_days': 5,
            'ideal_task_completion': 30
        }
    
    def analyze(self, features):
        """Analyze burnout risk based on workload patterns"""
        score = 0
        factors = []
        
        # Work hours analysis
        if features['avg_work_hours'] > self.thresholds['max_work_hours']:
            score += 30
            factors.append(f"Working {features['avg_work_hours']} hours/day on average")
        
        # Weekend work
        if features['weekend_work'] > self.thresholds['max_weekend_work']:
            score += 25
            factors.append(f"Working {features['weekend_work']} weekends/month")
        
        # Overtime
        if features['overtime_days'] > self.thresholds['max_overtime_days']:
            score += 25
            factors.append(f"{features['overtime_days']} overtime days detected")
        
        # Task overload
        if features['tasks_completed'] > self.thresholds['ideal_task_completion']:
            score += 20
            factors.append(f"{features['tasks_completed']} tasks completed (high load)")
        
        risk_level = 'high' if score >= 70 else 'medium' if score >= 40 else 'low'
        
        recommendations = self._generate_recommendations(risk_level, features)
        
        return {
            'risk_level': risk_level,
            'score': score,
            'factors': factors,
            'recommendations': recommendations
        }
    
    def _generate_recommendations(self, risk_level, features):
        recommendations = []
        if risk_level == 'high':
            recommendations.extend([
                'Immediate workload redistribution recommended',
                'Schedule mandatory time off',
                'Consider hiring additional team members'
            ])
        elif risk_level == 'medium':
            recommendations.extend([
                'Monitor workload closely',
                'Limit overtime to essential tasks only',
                'Encourage regular breaks'
            ])
        else:
            recommendations.append('Maintain current workload balance')
        
        return recommendations
```

## Configuration

### MongoDB Schema Examples

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
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
    enum: ['admin', 'user'],
    default: 'user'
  },
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  assignedTasks: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Task'
  }],
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: Date
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
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
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  dueDate: Date,
  timeSpent: {
    type: Number,
    default: 0 // in seconds
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

## Common Patterns

### Protected Route Pattern (Frontend)

```javascript
// src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const { user, loading } = useAuth();

  if (loading) {
    return <div>Loading...</div>;
  }

  if (!user) {
    return <Navigate to="/login" />;
  }

  if (requireAdmin && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';

<Routes>
  <Route path="/login" element={<Login />} />
  <Route path="/dashboard" element={
    <ProtectedRoute>
      <Dashboard />
    </ProtectedRoute>
  } />
  <Route path="/admin" element={
    <ProtectedRoute requireAdmin={true}>
      <AdminPanel />
    </ProtectedRoute>
  } />
</Routes>
```

### API Service Layer Pattern

```javascript
// src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_URL = process.env.REACT_APP_ML_SERVICE_URL;

const api = axios.create({
  baseURL: API_URL
});

const mlApi = axios.create({
  baseURL: ML_URL
});

// Auto-attach token
api.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const userService = {
  getAll: () => api.get('/api/users'),
  create: (data) => api.post('/api/users', data),
  update: (id, data) => api.put(`/api/users/${id}`, data),
  delete: (id) => api.delete(`/api/users/${id}`)
};

export const taskService = {
  getAll: () => api.get('/api/tasks'),
  create: (data) => api.post('/api/tasks', data),
  updateStatus: (id, status) => api.patch(`/api/tasks/${id}/status`, { status }),
  trackTime: (id, timeSpent) => api.post(`/api/tasks/${id}/time`, { timeSpent })
};

export const mlService = {
  predictRisk: (data) => mlApi.post('/api/ml/predict-risk', data),
  analyzeBurnout: (data) => mlApi.post('/api/ml/analyze-burnout', data),
  classifyTicket: (data) => mlApi.post('/api/ml/classify-ticket', data)
};
```

### Real-time Notifications Pattern

```javascript
// backend/utils/notifications.js
const User = require('../models/User');
const Notification = require('../models/Notification');

const createNotification = async (userId, type, message, metadata = {}) => {
  const notification = new Notification({
    user: userId,
    type,
    message,
    metadata,
    read: false
  });
  
  await notification.save();
  
  // Emit via WebSocket if connected
  const io = require('../server').io;
  io.to(userId.toString()).emit('notification', notification);
  
  return notification;
};

const notifyTaskAssignment = async (taskId, userId, taskTitle) => {
  return createNotification(
    userId,
    'task_assigned',
    `You have been assigned a new task: ${taskTitle}`,
    { taskId }
  );
};

const notifyRiskDetected = async (userId, riskLevel, factors) => {
  const admins = await User.find({ role: 'admin' });
  
  for (const admin of admins) {
    await createNotification(
      admin._id,
      'risk_alert',
      `Risk detected for user ${userId}: ${riskLevel} risk level`,
      { userId, riskLevel, factors }
    );
  }
};

module.exports = {
  createNotification,
  notifyTaskAssignment,
  notifyRiskDetected
};
```

## Troubleshooting

### JWT Token Expired

**Problem**: Getting 403 errors even after login.

**Solution**: Check token expiration and implement refresh token logic:

```javascript
// backend/routes/auth.js
const generateTokens = (userId) => {
  const accessToken = jwt.sign(
    { id: userId },
    process.env.JWT_SECRET,
    { expiresIn: '15m' }
  );
  
  const refreshToken = jwt.sign(
    { id: userId },
    process.env.JWT_REFRESH_SECRET,
    { expiresIn: '7d' }
  );
  
  return { accessToken, refreshToken };
};

router.post('/refresh', async (req, res) => {
  const { refreshToken } = req.body;
  
  try {
    const decoded = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET);
    const tokens = generateTokens(decoded.id);
    res.json(tokens);
  } catch (error) {
    res.status(403).json({ message: 'Invalid refresh token' });
  }
});
```

### MongoDB Connection Issues

**Problem**: Cannot connect to MongoDB.

**Solution**: Ensure MongoDB is running and connection string is correct:

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

### ML Model Not Loading

**Problem**: ML predictions failing with model not found errors.

**Solution**: Initialize models with fallback:

```python
# ml-service/models/risk_predictor.py
import os
from pathlib import Path

class RiskPredictor:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        
        # Ensure directory exists
        Path(os.path.dirname(model_path)).mkdir(
