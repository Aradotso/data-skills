---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics dashboard"
  - "create task management with burnout detection"
  - "build support ticket system with AI classification"
  - "add anomaly detection to user management"
  - "integrate ML service for risk prediction"
  - "develop kanban board with time tracking"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack JavaScript application that combines user/task management with AI-powered analytics. The system provides role-based access control (admin/user), kanban-style task tracking, support ticket management, and ML-driven insights including risk detection, anomaly detection, burnout analysis, and predictive project delays.

**Architecture:**
- Frontend: React.js (port 3000)
- Backend: Node.js with Express (port 5000)
- ML Service: FastAPI with scikit-learn and River (port 8000)
- Database: MongoDB
- Authentication: JWT

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
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
MONGODB_URI=${MONGODB_URI}
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
MONGO_URI=${MONGODB_URI}
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Backend API Reference

### Authentication Endpoints

```javascript
// User Registration
POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@company.com",
  "password": "securePass123",
  "role": "user" // or "admin"
}

// User Login
POST /api/auth/login
{
  "email": "john@company.com",
  "password": "securePass123"
}
// Returns: { token: "jwt_token", user: {...} }

// Get Current User
GET /api/auth/me
Headers: { Authorization: "Bearer <token>" }
```

### User Management (Admin Only)

```javascript
// Get All Users
GET /api/users
Headers: { Authorization: "Bearer <admin_token>" }

// Update User
PUT /api/users/:userId
Headers: { Authorization: "Bearer <admin_token>" }
{
  "role": "admin",
  "status": "active"
}

// Delete User
DELETE /api/users/:userId
Headers: { Authorization: "Bearer <admin_token>" }
```

### Task Management

```javascript
// Create Task
POST /api/tasks
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Implement AI feature",
  "description": "Add burnout detection",
  "assignedTo": "userId",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Get Tasks
GET /api/tasks?status=in-progress&assignedTo=userId
Headers: { Authorization: "Bearer <token>" }

// Update Task Status
PATCH /api/tasks/:taskId/status
{
  "status": "done"
}

// Track Time
POST /api/tasks/:taskId/time
{
  "duration": 3600 // seconds
}
```

### Support Ticket Management

```javascript
// Create Ticket
POST /api/tickets
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Login Issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// Get Tickets
GET /api/tickets?status=open&priority=high
Headers: { Authorization: "Bearer <token>" }

// Update Ticket
PATCH /api/tickets/:ticketId
{
  "status": "in-progress",
  "assignedTo": "adminUserId"
}
```

## ML Service API Reference

### Risk Detection

```javascript
// Predict User Risk Score
POST http://localhost:8000/predict/risk
{
  "userId": "user123",
  "loginAttempts": 5,
  "failedLogins": 2,
  "lastLoginTime": "2026-04-15T10:30:00Z",
  "tasksCompleted": 15,
  "tasksPending": 8,
  "avgTaskCompletionTime": 7200
}
// Returns: { riskScore: 0.65, riskLevel: "medium" }
```

### Anomaly Detection

```javascript
// Detect Anomalous Behavior
POST http://localhost:8000/detect/anomaly
{
  "userId": "user123",
  "action": "bulk_delete",
  "timestamp": "2026-04-15T23:45:00Z",
  "ipAddress": "192.168.1.100",
  "resourcesAccessed": 50
}
// Returns: { isAnomaly: true, confidence: 0.87 }
```

### Burnout Detection

```javascript
// Analyze User Burnout
POST http://localhost:8000/analyze/burnout
{
  "userId": "user123",
  "hoursWorked": 55,
  "tasksAssigned": 20,
  "tasksCompleted": 12,
  "overtimeHours": 15,
  "weekendWork": 8
}
// Returns: { burnoutScore: 0.78, recommendation: "reduce_workload" }
```

### Ticket Classification

```javascript
// Auto-classify Support Ticket
POST http://localhost:8000/classify/ticket
{
  "title": "Cannot access reports",
  "description": "Getting 403 error when trying to view analytics dashboard",
  "userRole": "user"
}
// Returns: { category: "access_control", priority: "high", suggestedAssignee: "admin_id" }
```

### Project Delay Prediction

```javascript
// Predict Project Delay
POST http://localhost:8000/predict/delay
{
  "projectId": "proj123",
  "totalTasks": 50,
  "completedTasks": 20,
  "daysElapsed": 30,
  "totalDays": 60,
  "teamSize": 5,
  "avgTaskDuration": 4800
}
// Returns: { delayProbability: 0.68, estimatedDelayDays: 12 }
```

## Frontend Integration Patterns

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
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(res.data.user);
    } catch (err) {
      logout();
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
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
    <AuthContext.Provider value={{ user, token, login, logout }}>
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
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const res = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/tasks?assignedTo=${userId}`
    );
    
    const grouped = {
      todo: res.data.filter(t => t.status === 'todo'),
      inProgress: res.data.filter(t => t.status === 'in-progress'),
      done: res.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await axios.patch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
      { status: newStatus }
    );
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select onChange={(e) => moveTask(task._id, e.target.value)} value={status}>
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
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
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
      // Fetch risk score
      const riskRes = await axios.post(
        `${process.env.REACT_APP_ML_URL}/predict/risk`,
        { userId }
      );

      // Fetch burnout analysis
      const burnoutRes = await axios.post(
        `${process.env.REACT_APP_ML_URL}/analyze/burnout`,
        { userId }
      );

      setAnalytics({
        riskScore: riskRes.data.riskScore,
        burnoutScore: burnoutRes.data.burnoutScore,
        recommendation: burnoutRes.data.recommendation
      });
    } catch (err) {
      console.error('Analytics fetch failed:', err);
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
        <div className={`score ${analytics.burnoutScore > 0.7 ? 'high' : 'low'}`}>
          {(analytics.burnoutScore * 100).toFixed(0)}%
        </div>
        {analytics.recommendation && (
          <p className="recommendation">{analytics.recommendation}</p>
        )}
      </div>
    </div>
  );
};

export default AIAnalytics;
```

### Time Tracking Hook

```javascript
// src/hooks/useTimeTracker.js
import { useState, useEffect, useRef } from 'react';
import axios from 'axios';

export const useTimeTracker = (taskId) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else {
      clearInterval(intervalRef.current);
    }
    return () => clearInterval(intervalRef.current);
  }, [isRunning]);

  const start = () => setIsRunning(true);
  
  const stop = async () => {
    setIsRunning(false);
    if (seconds > 0) {
      await axios.post(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`,
        { duration: seconds }
      );
    }
  };

  const reset = () => {
    setSeconds(0);
    setIsRunning(false);
  };

  return { seconds, isRunning, start, stop, reset };
};
```

## Backend Implementation Patterns

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
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
  } catch (err) {
    res.status(401).json({ message: 'Token invalid' });
  }
};

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: `Role ${req.user.role} not authorized` 
      });
    }
    next();
  };
};

module.exports = { protect, authorize };
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

    // Check for potential project delay
    const delayPrediction = await axios.post(
      `${process.env.ML_SERVICE_URL}/predict/delay`,
      { projectId: task.projectId }
    );

    res.status(201).json({ 
      task, 
      delayWarning: delayPrediction.data 
    });
  } catch (err) {
    res.status(400).json({ message: err.message });
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
  } catch (err) {
    res.status(400).json({ message: err.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    task.timeTracked = (task.timeTracked || 0) + req.body.duration;
    await task.save();
    res.json({ timeTracked: task.timeTracked });
  } catch (err) {
    res.status(400).json({ message: err.message });
  }
};
```

## ML Service Implementation

### Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
from river import tree
import numpy as np

class RiskPredictor:
    def __init__(self):
        self.model = tree.HoeffdingTreeClassifier()
        
    def extract_features(self, user_data):
        return np.array([
            user_data.get('loginAttempts', 0),
            user_data.get('failedLogins', 0),
            user_data.get('tasksCompleted', 0),
            user_data.get('tasksPending', 0),
            user_data.get('avgTaskCompletionTime', 0) / 3600
        ])
    
    def predict(self, user_data):
        features = self.extract_features(user_data)
        feature_dict = {f'f{i}': v for i, v in enumerate(features)}
        
        # Online prediction
        risk_proba = self.model.predict_proba_one(feature_dict)
        risk_score = risk_proba.get(1, 0.5)
        
        # Determine risk level
        if risk_score > 0.7:
            level = "high"
        elif risk_score > 0.4:
            level = "medium"
        else:
            level = "low"
            
        return {"riskScore": risk_score, "riskLevel": level}
    
    def update(self, user_data, actual_risk):
        features = self.extract_features(user_data)
        feature_dict = {f'f{i}': v for i, v in enumerate(features)}
        self.model.learn_one(feature_dict, actual_risk)
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.risk_predictor import RiskPredictor
from models.burnout_analyzer import BurnoutAnalyzer
from models.ticket_classifier import TicketClassifier
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Initialize models
risk_predictor = RiskPredictor()
burnout_analyzer = BurnoutAnalyzer()
ticket_classifier = TicketClassifier()

class RiskRequest(BaseModel):
    userId: str
    loginAttempts: int = 0
    failedLogins: int = 0
    tasksCompleted: int = 0
    tasksPending: int = 0
    avgTaskCompletionTime: float = 0

class BurnoutRequest(BaseModel):
    userId: str
    hoursWorked: float
    tasksAssigned: int
    tasksCompleted: int
    overtimeHours: float = 0
    weekendWork: float = 0

class TicketRequest(BaseModel):
    title: str
    description: str
    userRole: str = "user"

@app.post("/predict/risk")
async def predict_risk(request: RiskRequest):
    try:
        result = risk_predictor.predict(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze/burnout")
async def analyze_burnout(request: BurnoutRequest):
    try:
        result = burnout_analyzer.analyze(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/classify/ticket")
async def classify_ticket(request: TicketRequest):
    try:
        result = ticket_classifier.classify(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Configuration

### Backend Environment Variables

```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
RATE_LIMIT_WINDOW=15
RATE_LIMIT_MAX=100
```

### Frontend Environment Variables

```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_SOCKET_URL=http://localhost:5000
REACT_APP_MAX_FILE_SIZE=5242880
```

### ML Service Environment Variables

```bash
# ml-service/.env
MONGO_URI=mongodb://localhost:27017/enterprise_ums
MODEL_PATH=./models
LOG_LEVEL=INFO
BATCH_SIZE=32
PREDICTION_THRESHOLD=0.7
```

## Common Patterns

### Axios Instance Setup

```javascript
// frontend/src/utils/api.js
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

export default api;
```

### Protected Route Component

```javascript
// frontend/src/components/ProtectedRoute.jsx
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, token } = useContext(AuthContext);

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user?.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

## Troubleshooting

### JWT Token Expiration

```javascript
// Handle token refresh
const refreshToken = async () => {
  try {
    const res = await axios.post('/api/auth/refresh', {
      token: localStorage.getItem('refreshToken')
    });
    localStorage.setItem('token', res.data.token);
    return res.data.token;
  } catch (err) {
    // Force logout
    localStorage.clear();
    window.location.href = '/login';
  }
};
```

### ML Service Connection Issues

```javascript
// Backend fallback when ML service is down
const getPrediction = async (data) => {
  try {
    const res = await axios.post(
      `${process.env.ML_SERVICE_URL}/predict/risk`,
      data,
      { timeout: 5000 }
    );
    return res.data;
  } catch (err) {
    console.error('ML service unavailable, using fallback');
    // Return default low-risk prediction
    return { riskScore: 0.3, riskLevel: 'low' };
  }
};
```

### MongoDB Connection Error

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected');
  } catch (err) {
    console.error('MongoDB connection error:', err.message);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH']
}));
```

### Model Loading Issues

```python
# ml-service/utils/model_loader.py
import os
import pickle
import logging

logger = logging.getLogger(__name__)

def load_model(model_name):
    model_path = os.path.join(os.getenv('MODEL_PATH', './models'), f'{model_name}.pkl')
    
    if not os.path.exists(model_path):
        logger.warning(f"Model {model_name} not found, initializing new model")
        return None
    
    try:
        with open(model_path, 'rb') as f:
            return pickle.load(f)
    except Exception as e:
        logger.error(f"Error loading model {model_name}: {e}")
        return None
```

## Performance Optimization

### Task Query Optimization

```javascript
// backend/controllers/taskController.js
exports.getTasks = async (req, res) => {
  const { status, assignedTo, priority } = req.query;
  
  const query = {};
  if (status) query.status = status;
  if (assignedTo) query.assignedTo = assignedTo;
  if (priority) query.priority = priority;
  
  // Use lean() for faster queries when no mongoose methods needed
  const tasks = await Task.find(query)
    .lean()
    .select('-__v')
    .sort({ createdAt: -1 })
    .limit(100);
  
  res.json(tasks);
};
```

### Frontend State Caching

```javascript
// Use React Query for caching
import { useQuery } from 'react-query';

const useTasks = (userId) => {
  return useQuery(['tasks', userId], 
    () => api.get(`/api/tasks?assignedTo=${userId}`).then(res => res.data),
    { 
      staleTime: 30000, // 30 seconds
      cacheTime: 300000 // 5 minutes
    }
  );
};
```
