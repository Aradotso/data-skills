---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task automation using React, Node.js, and FastAPI ML service
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management dashboard with AI insights"
  - "implement task tracking with burnout detection"
  - "build admin panel with anomaly detection"
  - "add AI-based ticket classification system"
  - "integrate predictive analytics for user management"
  - "develop user management system with ML features"
  - "configure JWT authentication for user management app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-powered analytics. It provides admin controls, user dashboards, Kanban task boards, support ticket systems, and ML-based insights including risk prediction, anomaly detection, burnout analysis, and predictive project delays.

**Architecture:**
- Frontend: React.js (port 3000)
- Backend: Node.js REST API (port 5000)
- ML Service: FastAPI with scikit-learn and River (port 8000)
- Database: MongoDB
- Authentication: JWT tokens

## Installation

### Prerequisites

```bash
# Required software
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

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Backend API Reference

### Authentication Endpoints

```javascript
// Register new user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePass123"
}
// Returns: { token: "jwt_token", user: {...} }

// Get current user
GET /api/auth/me
Headers: { Authorization: "Bearer <token>" }
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer <admin_token>" }

// Update user
PUT /api/users/:userId
Headers: { Authorization: "Bearer <admin_token>" }
{
  "name": "Updated Name",
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:userId
Headers: { Authorization: "Bearer <admin_token>" }
```

### Task Management

```javascript
// Create task
POST /api/tasks
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Implement feature X",
  "description": "Build new dashboard component",
  "assignedTo": "userId",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:taskId/status
{
  "status": "in-progress" // or "done"
}

// Get user tasks
GET /api/tasks/user/:userId
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Cannot access dashboard",
  "description": "Getting 404 error",
  "priority": "high",
  "category": "technical"
}

// AI-classify ticket
POST /api/tickets/:ticketId/classify
// Returns: { category: "technical", priority: "high", assignedTo: "userId" }

// Update ticket
PATCH /api/tickets/:ticketId
{
  "status": "in-progress",
  "assignedTo": "supportUserId"
}
```

## ML Service API Reference

### Risk Prediction

```python
# Predict user risk score
POST /api/ml/risk-prediction
{
  "userId": "user123",
  "taskCount": 15,
  "completedTasks": 8,
  "overdueTask": 4,
  "loginFrequency": 20,
  "lastLogin": "2026-04-15T10:30:00Z"
}
# Returns: { riskScore: 0.75, riskLevel: "high", factors: [...] }
```

### Anomaly Detection

```python
# Detect anomalous behavior
POST /api/ml/anomaly-detection
{
  "userId": "user123",
  "activityData": {
    "loginTime": "03:45:00",
    "location": "unusual_ip",
    "actionsPerHour": 150,
    "failedLoginAttempts": 5
  }
}
# Returns: { isAnomaly: true, confidence: 0.92, alertLevel: "critical" }
```

### Burnout Detection

```python
# Analyze burnout risk
POST /api/ml/burnout-detection
{
  "userId": "user123",
  "workHours": [8, 10, 12, 14, 11, 9, 8],
  "taskLoad": 25,
  "overtimeHours": 15,
  "missedDeadlines": 3
}
# Returns: { burnoutScore: 0.68, status: "moderate", recommendations: [...] }
```

### Predictive Project Insights

```python
# Predict project delay
POST /api/ml/project-prediction
{
  "projectId": "proj123",
  "tasksTotal": 50,
  "tasksCompleted": 20,
  "daysRemaining": 10,
  "teamSize": 5,
  "velocity": 2.5
}
# Returns: { delayProbability: 0.82, estimatedDelay: 7, suggestedActions: [...] }
```

## Frontend Implementation Patterns

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

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
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(res.data);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
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

  return { user, loading, login, logout, isAdmin: user?.role === 'admin' };
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`);
    const grouped = {
      todo: res.data.filter(t => t.status === 'todo'),
      inProgress: res.data.filter(t => t.status === 'in-progress'),
      done: res.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await axios.patch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
      status: newStatus
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(column => (
        <div key={column} className="kanban-column">
          <h3>{column.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          {tasks[column].map(task => (
            <div key={task._id} className="task-card" draggable>
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select onChange={(e) => moveTask(task._id, e.target.value)} value={task.status}>
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

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutScore: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch risk prediction
      const riskRes = await axios.post(`${process.env.REACT_APP_ML_API_URL}/api/ml/risk-prediction`, {
        userId,
        // Add relevant user metrics
      });

      // Fetch burnout detection
      const burnoutRes = await axios.post(`${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection`, {
        userId,
        // Add work metrics
      });

      setAnalytics({
        riskScore: riskRes.data.riskScore,
        burnoutScore: burnoutRes.data.burnoutScore,
        anomalies: []
      });
    } catch (error) {
      console.error('Analytics fetch failed:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <div className="metric-card">
        <h3>Risk Score</h3>
        <div className={`score ${analytics.riskScore > 0.7 ? 'high' : 'low'}`}>
          {(analytics.riskScore * 100).toFixed(0)}%
        </div>
      </div>
      <div className="metric-card">
        <h3>Burnout Risk</h3>
        <div className={`score ${analytics.burnoutScore > 0.6 ? 'high' : 'low'}`}>
          {(analytics.burnoutScore * 100).toFixed(0)}%
        </div>
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Backend Implementation Patterns

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
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

const adminOnly = (req, res, next) => {
  if (req.user && req.user.role === 'admin') {
    next();
  } else {
    res.status(403).json({ message: 'Admin access required' });
  }
};

module.exports = { protect, adminOnly };
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user._id
    });
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.taskId,
      { status: req.body.status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
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
            model = RandomForestClassifier(n_estimators=100, random_state=42)
            return model
    
    def predict(self, features):
        """
        features: dict with keys:
        - task_count, completed_tasks, overdue_tasks
        - login_frequency, days_since_last_login
        """
        X = np.array([[
            features.get('task_count', 0),
            features.get('completed_tasks', 0),
            features.get('overdue_tasks', 0),
            features.get('login_frequency', 0),
            features.get('days_since_last_login', 0)
        ]])
        
        risk_score = self.model.predict_proba(X)[0][1] if hasattr(self.model, 'predict_proba') else 0.5
        
        risk_level = 'low'
        if risk_score > 0.7:
            risk_level = 'high'
        elif risk_score > 0.4:
            risk_level = 'medium'
        
        return {
            'risk_score': float(risk_score),
            'risk_level': risk_level,
            'factors': self._get_risk_factors(features)
        }
    
    def _get_risk_factors(self, features):
        factors = []
        if features.get('overdue_tasks', 0) > 3:
            factors.append('High number of overdue tasks')
        if features.get('days_since_last_login', 0) > 7:
            factors.append('Infrequent login activity')
        return factors
```

### FastAPI Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector
import logging

app = FastAPI()
risk_predictor = RiskPredictor()
burnout_detector = BurnoutDetector()

class RiskRequest(BaseModel):
    userId: str
    taskCount: int
    completedTasks: int
    overdueTask: int
    loginFrequency: int
    lastLogin: str

class BurnoutRequest(BaseModel):
    userId: str
    workHours: list
    taskLoad: int
    overtimeHours: int
    missedDeadlines: int

@app.post("/api/ml/risk-prediction")
async def predict_risk(request: RiskRequest):
    try:
        features = {
            'task_count': request.taskCount,
            'completed_tasks': request.completedTasks,
            'overdue_tasks': request.overdueTask,
            'login_frequency': request.loginFrequency,
            'days_since_last_login': 0  # Calculate from lastLogin
        }
        result = risk_predictor.predict(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutRequest):
    try:
        result = burnout_detector.analyze({
            'work_hours': request.workHours,
            'task_load': request.taskLoad,
            'overtime_hours': request.overtimeHours,
            'missed_deadlines': request.missedDeadlines
        })
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Database Models

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
  createdAt: { type: Date, default: Date.now },
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

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
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
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

## Configuration

### Environment Variables

**Backend (.env):**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
FRONTEND_URL=http://localhost:3000
NODE_ENV=development
```

**ML Service (.env):**
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
TRAINING_INTERVAL=86400
```

**Frontend (.env):**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Common Troubleshooting

### CORS Issues

```javascript
// backend/server.js - Add CORS middleware
const cors = require('cors');
app.use(cors({
  origin: process.env.FRONTEND_URL,
  credentials: true
}));
```

### JWT Token Expiration

```javascript
// Frontend axios interceptor
axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### ML Model Not Loading

```python
# Ensure model directory exists
import os
os.makedirs('./models', exist_ok=True)

# Initialize with dummy data if model doesn't exist
if not os.path.exists(model_path):
    X_dummy = np.random.rand(100, 5)
    y_dummy = np.random.randint(0, 2, 100)
    model.fit(X_dummy, y_dummy)
    joblib.dump(model, model_path)
```

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection failed:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Performance Optimization

```javascript
// Add indexes for frequent queries
taskSchema.index({ assignedTo: 1, status: 1 });
userSchema.index({ email: 1 });

// Implement pagination
exports.getTasks = async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 10;
  const skip = (page - 1) * limit;

  const tasks = await Task.find()
    .skip(skip)
    .limit(limit)
    .sort({ createdAt: -1 });
  
  res.json({ tasks, page, limit });
};
```
