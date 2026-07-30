---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user management dashboard with AI insights"
  - "integrate machine learning risk detection"
  - "build task management with burnout analysis"
  - "deploy user management system with FastAPI ML service"
  - "configure JWT authentication for enterprise app"
  - "add AI-powered ticket classification"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System is a full-stack JavaScript/Python application that combines traditional user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization-wide analytics and audit logs

**Architecture**: React frontend → Node.js/Express backend → FastAPI ML service → MongoDB

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
python >= 3.8
mongodb >= 4.x
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

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/user-management
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Runs on http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MODEL_PATH=./models
DB_CONNECTION=mongodb://localhost:27017/user-management
API_KEY=your_api_key_here
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# Runs on http://localhost:8000
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
# Runs on http://localhost:3000
```

## Key API Endpoints

### Authentication

```javascript
// POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "securePass123",
  "role": "user"
}

// POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePass123"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": { "id": "123", "username": "john.doe", "role": "user" }
}
```

### User Management

```javascript
// GET /api/users - Get all users (admin only)
// GET /api/users/:id - Get user by ID
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user (admin only)
```

### Task Management

```javascript
// POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Add new dashboard widget",
  "assignedTo": "userId123",
  "status": "todo",
  "priority": "high",
  "dueDate": "2026-05-01"
}

// PUT /api/tasks/:id
{
  "status": "in-progress",
  "timeSpent": 3600 // seconds
}

// GET /api/tasks/user/:userId - Get tasks for user
```

### Support Tickets

```javascript
// POST /api/tickets
{
  "title": "Cannot access reports",
  "description": "Getting 403 error when accessing reports page",
  "priority": "medium",
  "userId": "userId123"
}

// AI classification happens automatically via ML service
```

### ML/AI Endpoints

```python
# POST /ml/predict-risk
{
  "userId": "userId123",
  "recentActivity": {
    "failedLogins": 3,
    "dataAccessAtOddHours": true,
    "abnormalDataDownload": false
  }
}

# POST /ml/detect-burnout
{
  "userId": "userId123",
  "workload": {
    "tasksCompleted": 45,
    "averageHoursPerDay": 11.5,
    "weekendWork": true,
    "missedDeadlines": 2
  }
}

# POST /ml/classify-ticket
{
  "title": "Cannot login to system",
  "description": "Error message: Invalid credentials"
}

# POST /ml/predict-delay
{
  "projectId": "proj123",
  "tasksRemaining": 15,
  "velocity": 2.5,
  "daysUntilDeadline": 20
}
```

## Frontend Implementation Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
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
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
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

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Task Board Component

```javascript
// src/components/TaskBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`
      );
      const grouped = groupTasksByStatus(response.data);
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const groupTasksByStatus = (taskList) => {
    return taskList.reduce((acc, task) => {
      const status = task.status === 'in-progress' ? 'inProgress' : task.status;
      acc[status] = acc[status] || [];
      acc[status].push(task);
      return acc;
    }, { todo: [], inProgress: [], done: [] });
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="task-board">
      <Column title="To Do" tasks={tasks.todo} onDrop={(id) => updateTaskStatus(id, 'todo')} />
      <Column title="In Progress" tasks={tasks.inProgress} onDrop={(id) => updateTaskStatus(id, 'in-progress')} />
      <Column title="Done" tasks={tasks.done} onDrop={(id) => updateTaskStatus(id, 'done')} />
    </div>
  );
};
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAIAnalytics();
  }, [userId]);

  const fetchAIAnalytics = async () => {
    try {
      setLoading(true);
      const [riskData, burnoutData] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_API_URL}/ml/predict-risk`, { userId }),
        axios.post(`${process.env.REACT_APP_ML_API_URL}/ml/detect-burnout`, { userId })
      ]);

      setAnalytics({
        riskScore: riskData.data.riskScore,
        riskFactors: riskData.data.factors,
        burnoutScore: burnoutData.data.burnoutScore,
        burnoutWarning: burnoutData.data.warning
      });
    } catch (error) {
      console.error('Error fetching AI analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="risk-indicator">
        <h3>Risk Score: {analytics.riskScore}/100</h3>
        <ul>
          {analytics.riskFactors.map((factor, idx) => (
            <li key={idx}>{factor}</li>
          ))}
        </ul>
      </div>
      <div className="burnout-indicator">
        <h3>Burnout Score: {analytics.burnoutScore}/100</h3>
        {analytics.burnoutWarning && (
          <div className="warning">{analytics.burnoutWarning}</div>
        )}
      </div>
    </div>
  );
};
```

## Backend Implementation

### Express Server Setup

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Database connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
})
.then(() => console.log('MongoDB connected'))
.catch(err => console.error('MongoDB connection error:', err));

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/tasks', require('./routes/tasks'));
app.use('/api/tickets', require('./routes/tickets'));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ message: 'No token, authorization denied' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Token is not valid' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  isActive: { type: Boolean, default: true }
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

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
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
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in seconds
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict
import joblib
import numpy as np
from datetime import datetime

app = FastAPI(title="ML Analytics Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models (create placeholder if not exists)
try:
    risk_model = joblib.load('models/risk_model.pkl')
    burnout_model = joblib.load('models/burnout_model.pkl')
except:
    risk_model = None
    burnout_model = None

class RiskInput(BaseModel):
    userId: str
    recentActivity: Dict

class BurnoutInput(BaseModel):
    userId: str
    workload: Dict

class TicketInput(BaseModel):
    title: str
    description: str

class ProjectDelayInput(BaseModel):
    projectId: str
    tasksRemaining: int
    velocity: float
    daysUntilDeadline: int

@app.post("/ml/predict-risk")
async def predict_risk(data: RiskInput):
    """Predict security risk based on user activity"""
    try:
        activity = data.recentActivity
        
        # Feature extraction
        risk_score = 0
        factors = []
        
        if activity.get('failedLogins', 0) > 2:
            risk_score += 30
            factors.append("Multiple failed login attempts")
        
        if activity.get('dataAccessAtOddHours', False):
            risk_score += 25
            factors.append("Data access at unusual hours")
        
        if activity.get('abnormalDataDownload', False):
            risk_score += 35
            factors.append("Abnormal data download pattern")
        
        if activity.get('locationChange', False):
            risk_score += 20
            factors.append("Login from unusual location")
        
        return {
            "userId": data.userId,
            "riskScore": min(risk_score, 100),
            "factors": factors,
            "recommendation": "Monitor user activity" if risk_score > 50 else "Normal activity",
            "timestamp": datetime.now().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/detect-burnout")
async def detect_burnout(data: BurnoutInput):
    """Detect employee burnout based on workload metrics"""
    try:
        workload = data.workload
        
        burnout_score = 0
        warning = None
        
        # Calculate burnout indicators
        if workload.get('averageHoursPerDay', 0) > 10:
            burnout_score += 35
        
        if workload.get('weekendWork', False):
            burnout_score += 20
        
        if workload.get('missedDeadlines', 0) > 1:
            burnout_score += 25
        
        if workload.get('tasksCompleted', 0) > 40:
            burnout_score += 20
        
        if burnout_score > 60:
            warning = "High burnout risk detected. Consider workload redistribution."
        elif burnout_score > 40:
            warning = "Moderate burnout risk. Monitor workload balance."
        
        return {
            "userId": data.userId,
            "burnoutScore": min(burnout_score, 100),
            "warning": warning,
            "recommendations": [
                "Reduce overtime hours",
                "Delegate tasks",
                "Schedule breaks"
            ] if burnout_score > 50 else [],
            "timestamp": datetime.now().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/classify-ticket")
async def classify_ticket(data: TicketInput):
    """Classify support ticket and route to appropriate team"""
    try:
        text = f"{data.title} {data.description}".lower()
        
        # Simple keyword-based classification
        if any(word in text for word in ['login', 'password', 'access', 'authentication']):
            category = "Authentication"
            team = "Security Team"
        elif any(word in text for word in ['bug', 'error', 'crash', 'not working']):
            category = "Bug Report"
            team = "Development Team"
        elif any(word in text for word in ['feature', 'request', 'enhancement', 'add']):
            category = "Feature Request"
            team = "Product Team"
        elif any(word in text for word in ['data', 'report', 'export', 'import']):
            category = "Data Issue"
            team = "Data Team"
        else:
            category = "General Support"
            team = "Support Team"
        
        # Priority detection
        priority = "high" if any(word in text for word in ['urgent', 'critical', 'asap']) else "medium"
        
        return {
            "category": category,
            "assignedTeam": team,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/predict-delay")
async def predict_delay(data: ProjectDelayInput):
    """Predict project delay based on velocity and remaining work"""
    try:
        # Calculate estimated completion time
        days_needed = data.tasksRemaining / data.velocity if data.velocity > 0 else float('inf')
        
        delay_days = days_needed - data.daysUntilDeadline
        at_risk = delay_days > 0
        
        return {
            "projectId": data.projectId,
            "atRisk": at_risk,
            "estimatedDelayDays": max(0, delay_days),
            "completionProbability": min(1.0, data.daysUntilDeadline / days_needed) if days_needed > 0 else 1.0,
            "recommendations": [
                "Increase team capacity",
                "Reduce scope",
                "Extend deadline"
            ] if at_risk else ["On track"],
            "timestamp": datetime.now().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Configuration

### Environment Variables

**Backend (.env)**:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/user-management
JWT_SECRET=your_secure_jwt_secret_minimum_32_chars
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=production
ALLOWED_ORIGINS=http://localhost:3000,https://yourdomain.com
```

**Frontend (.env)**:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENV=development
```

**ML Service (.env)**:
```env
MODEL_PATH=./models
DB_CONNECTION=mongodb://localhost:27017/user-management
LOG_LEVEL=INFO
```

## Common Patterns

### Protected Routes

```javascript
// src/components/ProtectedRoute.js
import { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useContext(AuthContext);

  if (loading) return <div>Loading...</div>;
  if (!user) return <Navigate to="/login" />;
  if (adminOnly && user.role !== 'admin') return <Navigate to="/dashboard" />;

  return children;
};

export default ProtectedRoute;
```

### API Service Layer

```javascript
// src/services/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export const userService = {
  getAll: () => api.get('/api/users'),
  getById: (id) => api.get(`/api/users/${id}`),
  update: (id, data) => api.put(`/api/users/${id}`, data),
  delete: (id) => api.delete(`/api/users/${id}`)
};

export const taskService = {
  getByUser: (userId) => api.get(`/api/tasks/user/${userId}`),
  create: (data) => api.post('/api/tasks', data),
  update: (id, data) => api.put(`/api/tasks/${id}`, data),
  delete: (id) => api.delete(`/api/tasks/${id}`)
};

export default api;
```

## Troubleshooting

### JWT Token Expiration

```javascript
// Handle token refresh
const refreshToken = async () => {
  try {
    const response = await axios.post(
      `${process.env.REACT_APP_API_URL}/api/auth/refresh`,
      {},
      { headers: { 'Authorization': `Bearer ${localStorage.getItem('refreshToken')}` }}
    );
    localStorage.setItem('token', response.data.token);
    return response.data.token;
  } catch (error) {
    logout();
  }
};
```

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });
    console.log('MongoDB Connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected. Attempting to reconnect...');
});

module.exports = connectDB;
```

### ML Service Not Responding

```python
# Check ML service health
import requests

def check_ml_service():
    try:
        response = requests.get('http://localhost:8000/health', timeout=5)
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"ML Service unavailable: {e}")
        return None
```

### CORS Issues

```javascript
// backend/server.js
const corsOptions = {
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));
```

### Performance Optimization

```javascript
// Implement request caching
const cache = new Map();

const getCachedData = async (key, fetchFunction, ttl = 300000) => {
  if (cache.has(key)) {
    const { data, timestamp } = cache.get(key);
    if (Date.now() - timestamp < ttl) {
      return data;
    }
  }
  
  const data = await fetchFunction();
  cache.set(key, { data, timestamp: Date.now() });
  return data;
};
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI coding agents to assist developers in implementing, customizing, and troubleshooting this full-stack application.
