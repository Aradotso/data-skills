---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement user task tracking with AI insights"
  - "configure risk detection and anomaly detection"
  - "create admin dashboard with AI analytics"
  - "build user management system with JWT authentication"
  - "add AI-powered ticket classification"
  - "implement burnout detection for users"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack enterprise user management system that combines traditional CRUD operations with AI-powered analytics. It provides user authentication, task management, support ticket handling, and intelligent insights including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

## What This Project Does

The Enterprise User Management System with AI Analytics provides:

- **User Management**: Role-based access control with JWT authentication
- **Task Management**: Kanban board with time tracking and progress monitoring
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk detection, anomaly detection, burnout prediction, and project insights
- **Admin Dashboard**: Centralized control for user, task, and ticket management
- **User Dashboard**: Personal task overview with performance metrics

## Installation

### Prerequisites

- Node.js (v14+)
- Python 3.8+
- MongoDB

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
MONGO_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
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
REACT_APP_ML_API_URL=http://localhost:8000
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
// Returns: { token: "jwt_token", user: {...} }
```

### User Management

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { Authorization: "Bearer <token>" }

// Update user
PUT /api/users/:userId
Headers: { Authorization: "Bearer <token>" }
{
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:userId
Headers: { Authorization: "Bearer <token>" }
```

### Task Management

```javascript
// Create task
POST /api/tasks
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:taskId/status
{
  "status": "in-progress"
}

// Get user tasks
GET /api/tasks/user/:userId
Headers: { Authorization: "Bearer <token>" }
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { Authorization: "Bearer <token>" }
{
  "subject": "Login issue",
  "description": "Unable to login with correct credentials",
  "priority": "high"
}

// AI classify ticket
POST /api/tickets/:ticketId/classify
Headers: { Authorization: "Bearer <token>" }
// Returns: { category: "authentication", urgency: "high" }
```

### AI Analytics

```javascript
// Risk detection
POST /api/ai/risk-detection
Headers: { Authorization: "Bearer <token>" }
{
  "userId": "user_id",
  "activityData": {
    "loginAttempts": 5,
    "failedLogins": 3,
    "lastLogin": "2026-04-15T10:30:00Z"
  }
}

// Anomaly detection
POST /api/ai/anomaly-detection
{
  "userId": "user_id",
  "behavior": {
    "loginTime": "03:00",
    "location": "unusual_location",
    "device": "new_device"
  }
}

// Burnout analysis
POST /api/ai/burnout-detection
{
  "userId": "user_id",
  "workload": {
    "tasksCompleted": 45,
    "hoursWorked": 60,
    "overtimeHours": 20
  }
}

// Predictive insights
POST /api/ai/predictive-insights
{
  "projectId": "project_id",
  "metrics": {
    "completionRate": 0.65,
    "velocity": 8,
    "blockers": 3
  }
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

  const login = async (email, password) => {
    try {
      const response = await axios.post(`${API_URL}/api/auth/login`, {
        email,
        password
      });
      
      localStorage.setItem('token', response.data.token);
      setUser(response.data.user);
      return response.data;
    } catch (error) {
      throw error.response.data;
    }
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      // Verify token and get user
      axios.get(`${API_URL}/api/auth/me`, {
        headers: { Authorization: `Bearer ${token}` }
      })
      .then(res => setUser(res.data))
      .catch(() => logout())
      .finally(() => setLoading(false));
    } else {
      setLoading(false);
    }
  }, []);

  return { user, login, logout, loading };
};
```

### Task Board Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await axios.get(
      `${API_URL}/api/tasks/user/${userId}`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    
    const categorized = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    
    setTasks(categorized);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await axios.patch(
      `${API_URL}/api/tasks/${taskId}/status`,
      { status: newStatus },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    fetchTasks();
  };

  return (
    <div className="task-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="task-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status}
                onChange={(e) => updateTaskStatus(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in-progress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;
const API_URL = process.env.REACT_APP_API_URL;

const AIAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    anomalies: [],
    burnoutRisk: null,
    predictions: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    
    try {
      // Risk detection
      const riskResponse = await axios.post(
        `${API_URL}/api/ai/risk-detection`,
        { userId },
        { headers: { Authorization: `Bearer ${token}` } }
      );

      // Burnout detection
      const burnoutResponse = await axios.post(
        `${API_URL}/api/ai/burnout-detection`,
        { userId },
        { headers: { Authorization: `Bearer ${token}` } }
      );

      setAnalytics({
        riskScore: riskResponse.data.riskScore,
        anomalies: riskResponse.data.anomalies || [],
        burnoutRisk: burnoutResponse.data.burnoutRisk,
        predictions: burnoutResponse.data.recommendations || []
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <h2>AI Analytics Dashboard</h2>
      
      <div className="metric-card">
        <h3>Risk Score</h3>
        <div className={`score ${analytics.riskScore > 70 ? 'high' : 'low'}`}>
          {analytics.riskScore || 'N/A'}
        </div>
      </div>

      <div className="metric-card">
        <h3>Burnout Risk</h3>
        <div className={`risk ${analytics.burnoutRisk > 60 ? 'warning' : 'safe'}`}>
          {analytics.burnoutRisk ? `${analytics.burnoutRisk}%` : 'N/A'}
        </div>
      </div>

      <div className="anomalies-section">
        <h3>Detected Anomalies</h3>
        {analytics.anomalies.map((anomaly, idx) => (
          <div key={idx} className="anomaly-item">
            <span>{anomaly.type}</span>
            <span>{anomaly.severity}</span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Backend Implementation Examples

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true
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
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
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

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const JWT_SECRET = process.env.JWT_SECRET;

const authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const decoded = jwt.verify(token, JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user || user.status !== 'active') {
      return res.status(401).json({ error: 'Invalid authentication' });
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid authentication' });
  }
};

const isAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authenticate, isAdmin };
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
      createdBy: req.user._id,
      priority,
      dueDate,
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
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.taskId,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

## ML Service Implementation

### Risk Detection Model

```python
# ml-service/models/risk_detector.py
from river import tree, metrics
import numpy as np

class RiskDetector:
    def __init__(self):
        self.model = tree.HoeffdingTreeClassifier()
        self.metric = metrics.Accuracy()
        
    def predict_risk(self, features):
        """
        Features:
        - login_attempts: int
        - failed_logins: int
        - unusual_activity: bool
        - after_hours_access: bool
        - data_access_volume: float
        """
        feature_dict = {
            'login_attempts': features.get('login_attempts', 0),
            'failed_logins': features.get('failed_logins', 0),
            'unusual_activity': int(features.get('unusual_activity', False)),
            'after_hours_access': int(features.get('after_hours_access', False)),
            'data_access_volume': features.get('data_access_volume', 0.0)
        }
        
        prediction = self.model.predict_one(feature_dict)
        proba = self.model.predict_proba_one(feature_dict)
        
        risk_score = proba.get(1, 0) * 100 if proba else 0
        
        return {
            'risk_level': 'high' if risk_score > 70 else 'medium' if risk_score > 40 else 'low',
            'risk_score': round(risk_score, 2),
            'factors': self._identify_risk_factors(feature_dict)
        }
    
    def _identify_risk_factors(self, features):
        factors = []
        if features['failed_logins'] > 3:
            factors.append('Multiple failed login attempts')
        if features['unusual_activity'] == 1:
            factors.append('Unusual activity detected')
        if features['after_hours_access'] == 1:
            factors.append('After-hours access')
        return factors
```

### Burnout Detection

```python
# ml-service/models/burnout_detector.py
from river import linear_model, preprocessing
import numpy as np

class BurnoutDetector:
    def __init__(self):
        self.scaler = preprocessing.StandardScaler()
        self.model = linear_model.LinearRegression()
        
    def predict_burnout(self, workload_data):
        """
        workload_data:
        - tasks_completed: int
        - hours_worked: float
        - overtime_hours: float
        - weekend_work: bool
        - task_complexity_avg: float
        """
        features = {
            'tasks_completed': workload_data.get('tasks_completed', 0),
            'hours_worked': workload_data.get('hours_worked', 40),
            'overtime_hours': workload_data.get('overtime_hours', 0),
            'weekend_work': int(workload_data.get('weekend_work', False)),
            'task_complexity': workload_data.get('task_complexity_avg', 5)
        }
        
        # Normalize features
        normalized = {}
        for key, value in features.items():
            normalized[key] = self.scaler.transform_one({key: value})[key]
        
        # Calculate burnout risk
        burnout_score = self._calculate_burnout_score(features)
        
        return {
            'burnout_risk': round(burnout_score, 2),
            'risk_level': self._get_risk_level(burnout_score),
            'recommendations': self._get_recommendations(features, burnout_score)
        }
    
    def _calculate_burnout_score(self, features):
        score = 0
        
        # Hours worked weight
        if features['hours_worked'] > 50:
            score += 30
        elif features['hours_worked'] > 45:
            score += 15
            
        # Overtime weight
        score += min(features['overtime_hours'] * 2, 30)
        
        # Weekend work
        if features['weekend_work']:
            score += 20
            
        # Task load
        if features['tasks_completed'] > 40:
            score += 20
            
        return min(score, 100)
    
    def _get_risk_level(self, score):
        if score > 70:
            return 'critical'
        elif score > 50:
            return 'high'
        elif score > 30:
            return 'moderate'
        return 'low'
    
    def _get_recommendations(self, features, score):
        recommendations = []
        
        if features['overtime_hours'] > 10:
            recommendations.append('Reduce overtime hours')
        if features['weekend_work']:
            recommendations.append('Avoid weekend work')
        if features['hours_worked'] > 50:
            recommendations.append('Consider workload redistribution')
        if score > 70:
            recommendations.append('Schedule immediate break or vacation')
            
        return recommendations
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Dict, List, Optional
from models.risk_detector import RiskDetector
from models.burnout_detector import BurnoutDetector
import os

app = FastAPI(title="Enterprise User Management AI Service")

# Initialize models
risk_detector = RiskDetector()
burnout_detector = BurnoutDetector()

class RiskFeatures(BaseModel):
    login_attempts: int = 0
    failed_logins: int = 0
    unusual_activity: bool = False
    after_hours_access: bool = False
    data_access_volume: float = 0.0

class WorkloadData(BaseModel):
    tasks_completed: int
    hours_worked: float
    overtime_hours: float
    weekend_work: bool
    task_complexity_avg: float = 5.0

@app.post("/api/risk-detection")
async def detect_risk(features: RiskFeatures):
    try:
        result = risk_detector.predict_risk(features.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/burnout-detection")
async def detect_burnout(workload: WorkloadData):
    try:
        result = burnout_detector.predict_burnout(workload.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/anomaly-detection")
async def detect_anomaly(behavior: Dict):
    # Simple rule-based anomaly detection
    anomalies = []
    
    if behavior.get('login_time', '09:00') < '06:00' or behavior.get('login_time', '09:00') > '22:00':
        anomalies.append({
            'type': 'unusual_login_time',
            'severity': 'medium'
        })
    
    if behavior.get('new_device', False):
        anomalies.append({
            'type': 'new_device',
            'severity': 'high'
        })
    
    if behavior.get('unusual_location', False):
        anomalies.append({
            'type': 'unusual_location',
            'severity': 'high'
        })
    
    return {
        'anomalies': anomalies,
        'anomaly_score': len(anomalies) * 33.33,
        'requires_action': len(anomalies) > 1
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

## Common Patterns

### Protected Routes

```javascript
// frontend/src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useAuth();

  if (loading) {
    return <div>Loading...</div>;
  }

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

### Axios Interceptor for Auth

```javascript
// frontend/src/utils/axiosConfig.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const axiosInstance = axios.create({
  baseURL: API_URL
});

axiosInstance.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

axiosInstance.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default axiosInstance;
```

## Troubleshooting

### JWT Token Expiration

```javascript
// Implement token refresh
const refreshToken = async () => {
  try {
    const response = await axios.post('/api/auth/refresh', {
      refreshToken: localStorage.getItem('refreshToken')
    });
    localStorage.setItem('token', response.data.token);
    return response.data.token;
  } catch (error) {
    logout();
  }
};
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

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Service Not Responding

Check ML service logs and ensure dependencies are installed:

```bash
cd ml-service
pip install --upgrade -r requirements.txt
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Performance Optimization

```javascript
// Implement caching for analytics
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 300 }); // 5 minutes

app.get('/api/analytics/:userId', async (req, res) => {
  const cacheKey = `analytics_${req.params.userId}`;
  const cached = cache.get(cacheKey);
  
  if (cached) {
    return res.json(cached);
  }
  
  const analytics = await fetchAnalytics(req.params.userId);
  cache.set(cacheKey, analytics);
  res.json(analytics);
});
```

This skill provides comprehensive guidance for working with the Enterprise User Management System with AI Analytics, covering both the full-stack web application and the ML service components.
