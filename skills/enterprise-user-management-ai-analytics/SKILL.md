---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, anomaly detection, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement role-based access control with AI"
  - "create user dashboard with task tracking"
  - "add AI risk detection to user system"
  - "build admin panel with AI insights"
  - "deploy user management with ML service"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

The Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-powered analytics. It provides:

- **User Management**: Secure authentication, role-based access control, task tracking with Kanban boards
- **Admin Features**: User CRUD operations, task assignment, ticket management, audit logs
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, ticket classification, project delay prediction
- **ML Service**: FastAPI-based microservice using scikit-learn and River for real-time learning

The system consists of three main components: React frontend, Node.js/Express backend, and Python ML service.

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
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
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# or for development with hot reload
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

Application runs at `http://localhost:3000`

## Backend API Reference

### Authentication Endpoints

```javascript
// Register new user
POST /api/auth/register
Body: {
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
Body: {
  "email": "john@example.com",
  "password": "securepass123"
}
Response: {
  "token": "jwt_token_here",
  "user": { "id", "name", "email", "role" }
}

// Get current user
GET /api/auth/me
Headers: { "Authorization": "Bearer <token>" }
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <admin_token>" }

// Update user
PUT /api/users/:userId
Body: {
  "name": "Updated Name",
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:userId
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: { "Authorization": "Bearer <token>" }

// Create task (Admin)
POST /api/tasks
Body: {
  "title": "Implement feature X",
  "description": "Details here",
  "assignedTo": "userId",
  "priority": "high", // low, medium, high
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PUT /api/tasks/:taskId
Body: {
  "status": "in-progress" // todo, in-progress, done
}

// Track time
POST /api/tasks/:taskId/time
Body: {
  "duration": 3600 // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Body: {
  "title": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// Get user tickets
GET /api/tickets

// Update ticket (Admin)
PUT /api/tickets/:ticketId
Body: {
  "status": "resolved",
  "assignedTo": "adminUserId"
}
```

## ML Service API Reference

### Risk Detection

```javascript
// Analyze user risk
POST /ml/risk-detection
Body: {
  "userId": "user123",
  "loginAttempts": 5,
  "failedLogins": 2,
  "lastLoginTime": "2026-04-15T10:30:00Z",
  "activityPattern": "unusual",
  "dataAccessCount": 150
}
Response: {
  "riskScore": 0.75,
  "riskLevel": "high",
  "factors": ["Multiple failed logins", "Unusual activity pattern"],
  "recommendations": ["Enable 2FA", "Review access logs"]
}
```

### Anomaly Detection

```javascript
// Detect anomalous behavior
POST /ml/anomaly-detection
Body: {
  "userId": "user123",
  "features": {
    "loginFrequency": 45,
    "avgSessionDuration": 7200,
    "dataAccessRate": 180,
    "afterHoursActivity": 15
  }
}
Response: {
  "isAnomaly": true,
  "anomalyScore": 0.89,
  "explanation": "Unusually high after-hours activity"
}
```

### Burnout Analysis

```javascript
// Analyze user burnout risk
POST /ml/burnout-analysis
Body: {
  "userId": "user123",
  "tasksCompleted": 45,
  "avgTasksPerDay": 8,
  "overtimeHours": 25,
  "missedDeadlines": 3,
  "workloadTrend": "increasing"
}
Response: {
  "burnoutRisk": "high",
  "score": 0.82,
  "indicators": ["High overtime", "Increasing workload"],
  "recommendations": ["Redistribute tasks", "Schedule time off"]
}
```

### Ticket Classification

```javascript
// Classify support ticket
POST /ml/classify-ticket
Body: {
  "title": "Cannot reset password",
  "description": "Password reset email not arriving",
  "userHistory": ["previous_ticket_ids"]
}
Response: {
  "category": "technical",
  "priority": "high",
  "suggestedAssignee": "techSupportTeamId",
  "confidence": 0.92
}
```

### Predictive Insights

```javascript
// Predict project delay
POST /ml/project-insights
Body: {
  "projectId": "proj123",
  "tasksTotal": 50,
  "tasksCompleted": 20,
  "daysElapsed": 30,
  "daysRemaining": 20,
  "teamSize": 5,
  "blockers": 3
}
Response: {
  "delayProbability": 0.68,
  "estimatedDelay": 10, // days
  "completionDate": "2026-06-10",
  "risks": ["Multiple blockers", "Behind schedule"]
}
```

## Frontend Integration Patterns

### Setting Up API Client

```javascript
// src/utils/api.js
import axios from 'axios';

const API = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
});

// Add auth token to requests
API.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default API;
```

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';
import API from '../utils/api';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    checkAuth();
  }, []);

  const checkAuth = async () => {
    const token = localStorage.getItem('token');
    if (token) {
      try {
        const { data } = await API.get('/auth/me');
        setUser(data.user);
      } catch (error) {
        localStorage.removeItem('token');
      }
    }
    setLoading(false);
  };

  const login = async (email, password) => {
    const { data } = await API.post('/auth/login', { email, password });
    localStorage.setItem('token', data.token);
    setUser(data.user);
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### Task Management Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import API from '../utils/api';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const { data } = await API.get('/tasks');
      const grouped = {
        todo: data.filter(t => t.status === 'todo'),
        inProgress: data.filter(t => t.status === 'in-progress'),
        done: data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await API.put(`/tasks/${taskId}`, { status: newStatus });
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Failed to update task', error);
    }
  };

  return (
    <div className="task-board">
      <Column title="To Do" tasks={tasks.todo} onStatusChange={updateTaskStatus} />
      <Column title="In Progress" tasks={tasks.inProgress} onStatusChange={updateTaskStatus} />
      <Column title="Done" tasks={tasks.done} onStatusChange={updateTaskStatus} />
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Integration

```javascript
// src/services/aiService.js
import axios from 'axios';

const ML_API = axios.create({
  baseURL: process.env.REACT_APP_ML_SERVICE_URL,
});

export const analyzeUserRisk = async (userData) => {
  const { data } = await ML_API.post('/ml/risk-detection', userData);
  return data;
};

export const detectBurnout = async (workloadData) => {
  const { data } = await ML_API.post('/ml/burnout-analysis', workloadData);
  return data;
};

export const classifyTicket = async (ticketData) => {
  const { data } = await ML_API.post('/ml/classify-ticket', ticketData);
  return data;
};

// Usage in component
import { useEffect, useState } from 'react';
import { analyzeUserRisk } from '../services/aiService';

const RiskDashboard = ({ userId }) => {
  const [riskData, setRiskData] = useState(null);

  useEffect(() => {
    const fetchRisk = async () => {
      const userData = {
        userId,
        loginAttempts: 5,
        failedLogins: 1,
        lastLoginTime: new Date().toISOString(),
        activityPattern: 'normal',
        dataAccessCount: 100
      };
      const risk = await analyzeUserRisk(userData);
      setRiskData(risk);
    };
    fetchRisk();
  }, [userId]);

  if (!riskData) return <div>Loading risk analysis...</div>;

  return (
    <div className={`risk-level-${riskData.riskLevel}`}>
      <h3>Risk Score: {(riskData.riskScore * 100).toFixed(0)}%</h3>
      <ul>
        {riskData.factors.map((factor, i) => (
          <li key={i}>{factor}</li>
        ))}
      </ul>
    </div>
  );
};
```

## Backend Implementation Patterns

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  department: String,
  loginAttempts: { type: Number, default: 0 },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
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

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization?.startsWith('Bearer')) {
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
    res.status(401).json({ message: 'Invalid token' });
  }
};

exports.adminOnly = (req, res, next) => {
  if (req.user && req.user.role === 'admin') {
    next();
  } else {
    res.status(403).json({ message: 'Admin access required' });
  }
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.getTasks = async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user.id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .sort('-createdAt');
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    // Check permissions
    if (req.user.role !== 'admin' && 
        task.assignedTo.toString() !== req.user.id) {
      return res.status(403).json({ message: 'Not authorized' });
    }

    Object.assign(task, req.body);
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};
```

## ML Service Implementation

### Risk Detection Model

```python
# ml-service/models/risk_detector.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
from pathlib import Path

class RiskDetector:
    def __init__(self, model_path=None):
        self.model = None
        if model_path and Path(model_path).exists():
            self.model = joblib.load(model_path)
        else:
            self.model = RandomForestClassifier(n_estimators=100, random_state=42)
    
    def extract_features(self, user_data):
        """Extract features from user data"""
        return np.array([[
            user_data.get('loginAttempts', 0),
            user_data.get('failedLogins', 0),
            1 if user_data.get('activityPattern') == 'unusual' else 0,
            user_data.get('dataAccessCount', 0),
            user_data.get('afterHoursLogins', 0)
        ]])
    
    def predict_risk(self, user_data):
        """Predict risk score for user"""
        features = self.extract_features(user_data)
        
        if not hasattr(self.model, 'classes_'):
            # Model not trained, use rule-based approach
            return self._rule_based_risk(user_data)
        
        risk_proba = self.model.predict_proba(features)[0][1]
        return {
            'riskScore': float(risk_proba),
            'riskLevel': self._get_risk_level(risk_proba),
            'factors': self._identify_factors(user_data)
        }
    
    def _rule_based_risk(self, user_data):
        """Fallback rule-based risk assessment"""
        score = 0.0
        factors = []
        
        if user_data.get('failedLogins', 0) > 3:
            score += 0.3
            factors.append('Multiple failed logins')
        
        if user_data.get('activityPattern') == 'unusual':
            score += 0.4
            factors.append('Unusual activity pattern')
        
        if user_data.get('afterHoursLogins', 0) > 10:
            score += 0.2
            factors.append('High after-hours activity')
        
        return {
            'riskScore': min(score, 1.0),
            'riskLevel': self._get_risk_level(score),
            'factors': factors
        }
    
    def _get_risk_level(self, score):
        if score < 0.3:
            return 'low'
        elif score < 0.7:
            return 'medium'
        return 'high'
    
    def _identify_factors(self, user_data):
        factors = []
        if user_data.get('failedLogins', 0) > 2:
            factors.append('Multiple failed login attempts')
        if user_data.get('activityPattern') == 'unusual':
            factors.append('Unusual activity pattern detected')
        if user_data.get('dataAccessCount', 0) > 100:
            factors.append('High data access rate')
        return factors
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List
import os
from models.risk_detector import RiskDetector
from models.anomaly_detector import AnomalyDetector
from models.burnout_analyzer import BurnoutAnalyzer

app = FastAPI(title="Enterprise User Management ML Service")

# Initialize models
risk_detector = RiskDetector(model_path=os.getenv('MODEL_PATH'))
anomaly_detector = AnomalyDetector()
burnout_analyzer = BurnoutAnalyzer()

class RiskRequest(BaseModel):
    userId: str
    loginAttempts: int = 0
    failedLogins: int = 0
    lastLoginTime: str
    activityPattern: str = 'normal'
    dataAccessCount: int = 0
    afterHoursLogins: int = 0

class RiskResponse(BaseModel):
    riskScore: float
    riskLevel: str
    factors: List[str]
    recommendations: List[str]

@app.post("/ml/risk-detection", response_model=RiskResponse)
async def detect_risk(request: RiskRequest):
    """Analyze user risk based on behavioral patterns"""
    try:
        result = risk_detector.predict_risk(request.dict())
        
        # Generate recommendations
        recommendations = []
        if result['riskLevel'] in ['high', 'medium']:
            recommendations.append('Enable two-factor authentication')
            recommendations.append('Review recent activity logs')
        if result['riskScore'] > 0.7:
            recommendations.append('Consider temporary account restriction')
        
        return {
            **result,
            'recommendations': recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class BurnoutRequest(BaseModel):
    userId: str
    tasksCompleted: int
    avgTasksPerDay: float
    overtimeHours: int
    missedDeadlines: int
    workloadTrend: str

@app.post("/ml/burnout-analysis")
async def analyze_burnout(request: BurnoutRequest):
    """Analyze employee burnout risk"""
    try:
        result = burnout_analyzer.analyze(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class TicketRequest(BaseModel):
    title: str
    description: str
    userHistory: Optional[List[str]] = []

@app.post("/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    """Classify support ticket and suggest priority/assignment"""
    try:
        # Simple keyword-based classification
        text = f"{request.title} {request.description}".lower()
        
        category = 'general'
        priority = 'medium'
        
        if any(word in text for word in ['password', 'login', 'access', 'authentication']):
            category = 'technical'
            priority = 'high'
        elif any(word in text for word in ['bug', 'error', 'crash', 'not working']):
            category = 'technical'
            priority = 'high'
        elif any(word in text for word in ['feature', 'request', 'suggestion']):
            category = 'feature_request'
            priority = 'low'
        
        return {
            'category': category,
            'priority': priority,
            'suggestedAssignee': 'support-team',
            'confidence': 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Configuration

### Backend Environment Variables

```env
# Server
PORT=5000
NODE_ENV=production

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise-mgmt

# JWT
JWT_SECRET=your_secure_secret_key_here
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
```

### ML Service Environment Variables

```env
# Service
LOG_LEVEL=INFO
MODEL_PATH=./models

# Backend
BACKEND_URL=http://localhost:5000

# Model training (optional)
RETRAIN_INTERVAL=86400
MIN_SAMPLES_FOR_TRAINING=100
```

### Frontend Environment Variables

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
REACT_APP_ENABLE_AI_FEATURES=true
```

## Common Patterns

### Protected Route Component

```javascript
// src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  
  if (!user) return <Navigate to="/login" />;
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Task Time Tracker

```javascript
// src/components/TimeTracker.jsx
import { useState, useEffect } from 'react';
import API from '../utils/api';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

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
    if (seconds > 0) {
      await API.post(`/tasks/${taskId}/time`, { duration: seconds });
      setSeconds(0);
    }
  };

  const formatTime = (secs) => {
    const h = Math.floor(secs / 3600);
    const m = Math.floor((secs % 3600) / 60);
    const s = secs % 60;
    return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={handleStop} disabled={!seconds}>Save</button>
    </div>
  );
};
```

### Real-time Notifications

```javascript
// src/hooks/useNotifications.js
import { useEffect, useState } from 'react';
import API from '../utils/api';

export const useNotifications = () => {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    fetchNotifications();
    const interval = setInterval(fetchNotifications, 30000); // Poll every 30s
    return () => clearInterval(interval);
  }, []);

  const fetchNotifications = async () => {
    try {
      const { data } = await API.get('/notifications');
      setNotifications(data);
    } catch (error) {
      console.error('Failed to fetch notifications', error);
    }
  };

  const markAsRead = async (notificationId) => {
    try {
      await API.put(`/notifications/${notificationId}/read`);
      setNotifications(notifications.filter(n => n.id !== notificationId));
    } catch (error) {
      console.error('Failed to mark as read', error);
    }
  };

  return { notifications, markAsRead };
};
```

## Troubleshooting

### JWT Token Expired

**Problem**: 401 Unauthorized errors after some time

**Solution**:
```javascript
// Add token refresh logic
API.interceptors.response.use(
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

**Problem**: "MongoNetworkError: failed to connect"

**Solution**:
```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};
```

### ML Service CORS Errors

**Problem**: Frontend can't access ML service due to CORS

**Solution**:
```python
# ml-service/main.py
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Task Board Not Updating

**Problem**: Tasks don't move between columns

**Solution**:
```javascript
// Ensure proper state update
const updateTaskStatus = async (taskId, newStatus) => {
  try {
    await API.put(`/tasks/${taskId}`, { status: newStatus });
    // Optimistic update
    setTasks(prevTasks => {
      const allTasks = [...prevTasks.todo, ...prevTasks.inProgress, ...prevTasks.done];
      const task = allTasks.find(t => t.id === taskId);
      if (task) task.status = newStatus;
      return {
        todo: allTasks.filter(t => t.status === 'todo'),
        inProgress: allTasks.filter(t => t.status === 'in-progress'),
        done: allTasks.filter(t => t.status === 'done')
      };
    });
  } catch (error) {
    console.error('Update failed', error);
    fetchTasks(); // Reload on error
  }
};
```
