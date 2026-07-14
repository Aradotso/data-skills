---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, task management, and organizational insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "create user dashboard with task tracking"
  - "implement AI-powered ticket classification"
  - "build admin panel with user management"
  - "add burnout detection and risk prediction"
  - "configure JWT authentication for user system"
  - "deploy user management with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication, user CRUD operations
- **Task Management**: Kanban board, time tracking, assignment workflows
- **Support System**: Ticket management with AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization-wide analytics, audit logs, alerts

The system consists of three main components:
- **Frontend**: React.js application (port 3000)
- **Backend**: Node.js/Express REST API (port 5000)
- **ML Service**: FastAPI with scikit-learn and River (port 8000)

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB running locally or connection string
mongod --version
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../frontend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt
```

### Environment Configuration

**backend/.env**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**frontend/.env**
```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

**ml-service/.env**
```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
CORS_ORIGINS=http://localhost:3000,http://localhost:5000
```

## Running the Application

### Start All Services

```bash
# Terminal 1 - Backend
cd backend
npm start

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload --port 8000

# Terminal 3 - Frontend
cd frontend
npm start
```

### Production Build

```bash
# Build frontend
cd frontend
npm run build

# Serve with backend
cd ../backend
npm run production
```

## Key API Endpoints

### Authentication

```javascript
// POST /api/auth/register
const register = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
const login = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

```javascript
// GET /api/users - Get all users
const getUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management

```javascript
// GET /api/tasks - Get user tasks
const getTasks = async (token, filters = {}) => {
  const queryParams = new URLSearchParams(filters).toString();
  const response = await fetch(`http://localhost:5000/api/tasks?${queryParams}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: taskData.status, // 'todo', 'in-progress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, newStatus, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};
```

### Support Tickets

```javascript
// POST /api/tickets - Create ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      category: ticketData.category, // 'technical', 'billing', 'general'
      priority: ticketData.priority
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets (admin) or user tickets
const getTickets = async (token, isAdmin = false) => {
  const endpoint = isAdmin ? '/api/tickets/all' : '/api/tickets';
  const response = await fetch(`http://localhost:5000${endpoint}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## AI/ML Service API

### Risk Prediction

```javascript
// POST /predict/risk - Predict user risk score
const predictRisk = async (userId, token) => {
  const response = await fetch('http://localhost:8000/predict/risk', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      features: {
        login_frequency: 45,
        failed_login_attempts: 2,
        task_completion_rate: 0.85,
        avg_task_delay: 1.5,
        ticket_count: 3
      }
    })
  });
  return response.json();
  // Returns: { risk_score: 0.23, risk_level: 'low', factors: [...] }
};
```

### Anomaly Detection

```javascript
// POST /detect/anomaly - Detect unusual behavior
const detectAnomaly = async (userActivity, token) => {
  const response = await fetch('http://localhost:8000/detect/anomaly', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userActivity.userId,
      timestamp: new Date().toISOString(),
      activity_type: userActivity.type,
      metrics: {
        login_time: userActivity.loginTime,
        ip_address: userActivity.ipAddress,
        location: userActivity.location,
        device: userActivity.device
      }
    })
  });
  return response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.87, alert: true }
};
```

### Burnout Detection

```javascript
// POST /analyze/burnout - Analyze user burnout risk
const analyzeBurnout = async (userId, token) => {
  const response = await fetch('http://localhost:8000/analyze/burnout', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      workload_metrics: {
        active_tasks: 12,
        overdue_tasks: 3,
        avg_working_hours: 9.5,
        weekend_work_frequency: 0.4,
        task_switching_rate: 8
      }
    })
  });
  return response.json();
  // Returns: { burnout_risk: 'high', score: 0.78, recommendations: [...] }
};
```

### Ticket Classification

```javascript
// POST /classify/ticket - Auto-classify support ticket
const classifyTicket = async (ticketText, token) => {
  const response = await fetch('http://localhost:8000/classify/ticket', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketText.subject,
      description: ticketText.description
    })
  });
  return response.json();
  // Returns: { category: 'technical', priority: 'high', suggested_assignee: 'team_id' }
};
```

### Project Insights

```javascript
// POST /predict/project-delay - Predict project delays
const predictProjectDelay = async (projectData, token) => {
  const response = await fetch('http://localhost:8000/predict/project-delay', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      project_id: projectData.id,
      features: {
        total_tasks: 45,
        completed_tasks: 20,
        overdue_tasks: 5,
        avg_completion_time: 3.2,
        team_size: 5,
        days_remaining: 15
      }
    })
  });
  return response.json();
  // Returns: { delay_probability: 0.65, estimated_delay_days: 7, risk_factors: [...] }
};
```

## React Component Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      // Verify token and load user
      fetch('http://localhost:5000/api/auth/verify', {
        headers: { 'Authorization': `Bearer ${token}` }
      })
        .then(res => res.json())
        .then(data => {
          setUser(data.user);
          setLoading(false);
        })
        .catch(() => {
          localStorage.removeItem('token');
          setToken(null);
          setLoading(false);
        });
    } else {
      setLoading(false);
    }
  }, [token]);

  const login = async (credentials) => {
    const response = await fetch('http://localhost:5000/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(credentials)
    });
    const data = await response.json();
    
    if (data.token) {
      localStorage.setItem('token', data.token);
      setToken(data.token);
      setUser(data.user);
    }
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const KanbanBoard = () => {
  const { token } = useContext(AuthContext);
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch('http://localhost:5000/api/tasks', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    // Group tasks by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      'in-progress': data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks(); // Refresh
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    moveTask(taskId, newStatus);
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
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

export default KanbanBoard;
```

### Admin Dashboard with Analytics

```javascript
// src/components/AdminDashboard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const AdminDashboard = () => {
  const { token } = useContext(AuthContext);
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeUsers: 0,
    totalTasks: 0,
    completedTasks: 0,
    openTickets: 0,
    highRiskUsers: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    // Fetch overall statistics
    const usersRes = await fetch('http://localhost:5000/api/users/stats', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const tasksRes = await fetch('http://localhost:5000/api/tasks/stats', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const ticketsRes = await fetch('http://localhost:5000/api/tickets/stats', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    
    // Fetch AI risk predictions
    const riskRes = await fetch('http://localhost:8000/analyze/high-risk-users', {
      headers: { 'Authorization': `Bearer ${token}` }
    });

    const [users, tasks, tickets, risks] = await Promise.all([
      usersRes.json(),
      tasksRes.json(),
      ticketsRes.json(),
      riskRes.json()
    ]);

    setAnalytics({
      totalUsers: users.total,
      activeUsers: users.active,
      totalTasks: tasks.total,
      completedTasks: tasks.completed,
      openTickets: tickets.open,
      highRiskUsers: risks.users
    });
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-value">{analytics.totalUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Active Users</h3>
          <p className="stat-value">{analytics.activeUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Task Completion</h3>
          <p className="stat-value">
            {analytics.totalTasks > 0 
              ? Math.round((analytics.completedTasks / analytics.totalTasks) * 100)
              : 0}%
          </p>
        </div>
        <div className="stat-card alert">
          <h3>Open Tickets</h3>
          <p className="stat-value">{analytics.openTickets}</p>
        </div>
      </div>

      <div className="risk-alerts">
        <h2>High Risk Users</h2>
        {analytics.highRiskUsers.map(user => (
          <div key={user.id} className="risk-card">
            <h4>{user.name}</h4>
            <p>Risk Score: {(user.risk_score * 100).toFixed(1)}%</p>
            <p>Factors: {user.risk_factors.join(', ')}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
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
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Method to compare password
userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
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
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
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
  completedAt: Date,
  timeTracked: {
    type: Number,
    default: 0 // in seconds
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'bug', 'feature'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  aiClassification: {
    suggestedCategory: String,
    suggestedPriority: String,
    confidence: Number
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authenticate = async (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId).select('-password');
    
    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }

    req.user = user;
    req.userId = user._id;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid or expired token' });
  }
};

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        error: 'Insufficient permissions' 
      });
    }
    next();
  };
};

module.exports = { authenticate, authorize };
```

### ML Service Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=os.getenv("CORS_ORIGINS", "http://localhost:3000").split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models storage
models = {}

# Load or initialize models
def load_models():
    model_path = os.getenv("MODEL_PATH", "./models")
    os.makedirs(model_path, exist_ok=True)
    
    try:
        models['risk'] = joblib.load(f"{model_path}/risk_model.pkl")
        models['anomaly'] = joblib.load(f"{model_path}/anomaly_model.pkl")
    except:
        # Initialize new models if not found
        models['risk'] = RandomForestClassifier(n_estimators=100)
        models['anomaly'] = IsolationForest(contamination=0.1)

load_models()

# Request models
class RiskPredictionRequest(BaseModel):
    user_id: str
    features: Dict[str, float]

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    timestamp: str
    activity_type: str
    metrics: Dict[str, any]

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    workload_metrics: Dict[str, float]

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str

# Risk prediction endpoint
@app.post("/predict/risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Extract features
        features = np.array([
            request.features.get('login_frequency', 0),
            request.features.get('failed_login_attempts', 0),
            request.features.get('task_completion_rate', 0),
            request.features.get('avg_task_delay', 0),
            request.features.get('ticket_count', 0)
        ]).reshape(1, -1)
        
        # Predict (if model is trained, otherwise return mock)
        try:
            risk_score = models['risk'].predict_proba(features)[0][1]
        except:
            # Mock prediction if model not trained
            risk_score = float(np.mean(features) / 10)
        
        risk_level = 'high' if risk_score > 0.7 else 'medium' if risk_score > 0.4 else 'low'
        
        return {
            "user_id": request.user_id,
            "risk_score": float(risk_score),
            "risk_level": risk_level,
            "factors": [
                {"name": "Login Pattern", "impact": 0.3},
                {"name": "Task Completion", "impact": 0.4},
                {"name": "Support Tickets", "impact": 0.3}
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Anomaly detection endpoint
@app.post("/detect/anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        # Extract metrics as features
        features = np.array([
            hash(request.metrics.get('ip_address', '')) % 1000,
            hash(request.metrics.get('location', '')) % 1000,
            hash(request.metrics.get('device', '')) % 1000,
            hash(request.activity_type) % 1000
        ]).reshape(1, -1)
        
        # Predict anomaly
        try:
            prediction = models['anomaly'].predict(features)[0]
            anomaly_score = abs(models['anomaly'].score_samples(features)[0])
        except:
            # Mock prediction
            anomaly_score = float(np.random.random())
            prediction = -1 if anomaly_score > 0.7 else 1
        
        is_anomaly = prediction == -1
        
        return {
            "user_id": request.user_id,
            "is_anomaly": bool(is_anomaly),
            "anomaly_score": float(anomaly_score),
            "alert": is_anomaly and anomaly_score > 0.8,
            "timestamp": request.timestamp
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout analysis endpoint
@app.post("/analyze/burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    try:
        metrics = request.workload_metrics
        
        # Calculate burnout score
        score = (
            (metrics.get('active_tasks', 0) / 20) * 0.25 +
            (metrics.get('overdue_tasks', 0) / 10) * 0.3 +
            ((metrics.get('avg_working_hours', 0) - 8) / 8) * 0.2 +
            metrics.get('weekend_work_frequency', 0) * 0.15 +
            (metrics.get('task_switching_rate', 0) / 15) * 0.1
        )
        
        score = min
