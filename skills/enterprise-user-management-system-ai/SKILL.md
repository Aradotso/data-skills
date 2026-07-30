---
name: enterprise-user-management-system-ai
description: Full-stack user management system with AI-powered analytics, risk detection, and task management using React, Node.js, MongoDB, and FastAPI ML services
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement task tracking with burnout detection"
  - "create admin dashboard with anomaly detection"
  - "build support ticket system with AI classification"
  - "add risk prediction to user management"
  - "configure kanban board with time tracking"
  - "deploy user management system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-powered analytics. It provides role-based access control, Kanban task boards, support ticket management, and ML features including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

**Architecture:**
- **Frontend**: React.js dashboard (admin + user views)
- **Backend**: Node.js/Express REST API with JWT authentication
- **ML Service**: FastAPI with scikit-learn and River for online learning
- **Database**: MongoDB for user, task, and ticket data

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

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
EOF

npm start
# Backend runs at http://localhost:5000
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

uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
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

npm start
# Frontend runs at http://localhost:3000
```

## Core API Endpoints

### Authentication

```javascript
// Register user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securepass123"
}
// Returns: { token, user }
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer ${JWT_TOKEN}" }

// Update user
PUT /api/users/:id
{
  "name": "Updated Name",
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
  "description": "Add new functionality",
  "assignedTo": "userId",
  "priority": "high", // low, medium, high
  "status": "todo", // todo, in-progress, done
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:id/time
{
  "duration": 3600, // seconds
  "date": "2026-04-20"
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "subject": "Login issues",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// AI-powered ticket classification
POST /api/tickets/:id/classify
// Returns: { category, priority, suggestedAssignee }
```

## ML Service API

### Risk Prediction

```python
# FastAPI endpoint
POST /ml/predict-risk
{
  "userId": "123",
  "features": {
    "taskCount": 15,
    "overdueCount": 3,
    "avgCompletionTime": 72,
    "lastActivityDays": 2
  }
}
# Returns: { riskScore: 0.75, riskLevel: "high" }
```

### Anomaly Detection

```python
POST /ml/detect-anomaly
{
  "userId": "123",
  "activityData": {
    "loginTime": "2026-04-20T03:00:00Z",
    "location": "unusual-location",
    "deviceId": "new-device"
  }
}
# Returns: { isAnomaly: true, confidence: 0.89 }
```

### Burnout Analysis

```python
POST /ml/analyze-burnout
{
  "userId": "123",
  "workloadData": {
    "tasksAssigned": 25,
    "avgHoursPerDay": 12,
    "consecutiveDays": 14,
    "taskCompletionRate": 0.45
  }
}
# Returns: { burnoutRisk: 0.82, recommendations: [...] }
```

### Project Delay Prediction

```python
POST /ml/predict-delay
{
  "projectId": "proj-123",
  "features": {
    "tasksTotal": 50,
    "tasksCompleted": 15,
    "daysRemaining": 30,
    "teamSize": 5,
    "avgVelocity": 2.5
  }
}
# Returns: { delayProbability: 0.68, estimatedDelay: 15 }
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
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Axios Interceptor Setup

```javascript
// src/utils/axios.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

api.interceptors.request.use(
  config => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  error => Promise.reject(error)
);

api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import api from '../utils/axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const res = await api.get('/api/tasks');
    const grouped = res.data.reduce((acc, task) => {
      acc[task.status].push(task);
      return acc;
    }, { todo: [], 'in-progress': [], done: [] });
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await api.patch(`/api/tasks/${taskId}/status`, { status: newStatus });
    fetchTasks();
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = async (e, status) => {
    const taskId = e.dataTransfer.getData('taskId');
    await updateTaskStatus(taskId, status);
  };

  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={e => handleDrop(e, status)}
          onDragOver={e => e.preventDefault()}
        >
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div
              key={task._id}
              draggable
              onDragStart={e => handleDragStart(e, task._id)}
              className="task-card"
            >
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
// src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import api from '../utils/axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({});
  const mlApi = axios.create({ baseURL: process.env.REACT_APP_ML_API_URL });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch user data
      const userData = await api.get(`/api/users/${userId}`);
      const tasks = await api.get(`/api/tasks?userId=${userId}`);

      // Risk prediction
      const riskRes = await mlApi.post('/ml/predict-risk', {
        userId,
        features: {
          taskCount: tasks.data.length,
          overdueCount: tasks.data.filter(t => new Date(t.dueDate) < new Date()).length,
          avgCompletionTime: calculateAvgTime(tasks.data),
          lastActivityDays: daysSinceLastActivity(userData.data)
        }
      });

      // Burnout analysis
      const burnoutRes = await mlApi.post('/ml/analyze-burnout', {
        userId,
        workloadData: {
          tasksAssigned: tasks.data.length,
          avgHoursPerDay: calculateAvgHours(tasks.data),
          consecutiveDays: calculateConsecutiveDays(tasks.data),
          taskCompletionRate: calculateCompletionRate(tasks.data)
        }
      });

      setAnalytics({
        risk: riskRes.data,
        burnout: burnoutRes.data
      });
    } catch (error) {
      console.error('Analytics fetch failed:', error);
    }
  };

  const calculateAvgTime = (tasks) => {
    const completed = tasks.filter(t => t.status === 'done');
    if (!completed.length) return 0;
    const totalHours = completed.reduce((sum, t) => sum + (t.timeTracked || 0), 0);
    return totalHours / completed.length;
  };

  const calculateAvgHours = (tasks) => {
    const last7Days = tasks.filter(t => {
      const daysDiff = (new Date() - new Date(t.createdAt)) / (1000 * 60 * 60 * 24);
      return daysDiff <= 7;
    });
    return last7Days.reduce((sum, t) => sum + (t.timeTracked || 0), 0) / 7;
  };

  const calculateConsecutiveDays = (tasks) => {
    // Implementation for consecutive work days
    return 10; // Placeholder
  };

  const calculateCompletionRate = (tasks) => {
    const completed = tasks.filter(t => t.status === 'done').length;
    return tasks.length ? completed / tasks.length : 0;
  };

  const daysSinceLastActivity = (user) => {
    return Math.floor((new Date() - new Date(user.lastActive)) / (1000 * 60 * 60 * 24));
  };

  return (
    <div className="ai-analytics">
      <div className="risk-card">
        <h3>Risk Assessment</h3>
        <div className={`risk-level risk-${analytics.risk?.riskLevel}`}>
          {analytics.risk?.riskLevel?.toUpperCase()}
        </div>
        <p>Score: {analytics.risk?.riskScore?.toFixed(2)}</p>
      </div>

      <div className="burnout-card">
        <h3>Burnout Analysis</h3>
        <progress value={analytics.burnout?.burnoutRisk} max="1" />
        <p>Risk: {(analytics.burnout?.burnoutRisk * 100)?.toFixed(0)}%</p>
        {analytics.burnout?.recommendations?.map((rec, i) => (
          <li key={i}>{rec}</li>
        ))}
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Backend Implementation Patterns

### User Model (MongoDB/Mongoose)

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
  lastActive: { type: Date, default: Date.now },
  metadata: {
    loginCount: { type: Number, default: 0 },
    lastLoginIP: String,
    deviceId: String
  }
}, { timestamps: true });

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
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeTracked: { type: Number, default: 0 }, // in seconds
  timeEntries: [{
    duration: Number,
    date: Date,
    note: String
  }]
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

### JWT Middleware

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
    return res.status(401).json({ message: 'Token invalid' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: 'Not authorized for this action' });
    }
    next();
  };
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.getTasks = async (req, res) => {
  try {
    const filter = req.user.role === 'admin' ? {} : { assignedTo: req.user.id };
    const tasks = await Task.find(filter)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name')
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

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status },
      { new: true, runValidators: true }
    );
    if (!task) return res.status(404).json({ message: 'Task not found' });
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.addTimeEntry = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    if (!task) return res.status(404).json({ message: 'Task not found' });

    task.timeEntries.push(req.body);
    task.timeTracked += req.body.duration;
    await task.save();

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
from river import linear_model, preprocessing, compose
import pickle
import os

class RiskPredictor:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        self.model = self._load_or_create_model()
    
    def _load_or_create_model(self):
        if os.path.exists(self.model_path):
            with open(self.model_path, 'rb') as f:
                return pickle.load(f)
        return compose.Pipeline(
            preprocessing.StandardScaler(),
            linear_model.LogisticRegression()
        )
    
    def predict(self, features):
        """
        features: dict with taskCount, overdueCount, avgCompletionTime, lastActivityDays
        """
        score = self.model.predict_proba_one(features).get(1, 0.5)
        
        if score > 0.7:
            risk_level = "high"
        elif score > 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return {"riskScore": score, "riskLevel": risk_level}
    
    def train(self, features, label):
        """Online learning - update model with new data"""
        self.model.learn_one(features, label)
        self._save_model()
    
    def _save_model(self):
        os.makedirs(os.path.dirname(self.model_path), exist_ok=True)
        with open(self.model_path, 'wb') as f:
            pickle.dump(self.model, f)
```

### Burnout Analyzer

```python
# ml-service/models/burnout_analyzer.py
import numpy as np

class BurnoutAnalyzer:
    def analyze(self, workload_data):
        """
        workload_data: dict with tasksAssigned, avgHoursPerDay, 
                      consecutiveDays, taskCompletionRate
        """
        weights = {
            'task_pressure': 0.3,
            'hours_pressure': 0.3,
            'duration_pressure': 0.2,
            'completion_pressure': 0.2
        }
        
        task_score = min(workload_data['tasksAssigned'] / 30.0, 1.0)
        hours_score = min(workload_data['avgHoursPerDay'] / 12.0, 1.0)
        duration_score = min(workload_data['consecutiveDays'] / 21.0, 1.0)
        completion_score = 1.0 - workload_data['taskCompletionRate']
        
        burnout_risk = (
            task_score * weights['task_pressure'] +
            hours_score * weights['hours_pressure'] +
            duration_score * weights['duration_pressure'] +
            completion_score * weights['completion_pressure']
        )
        
        recommendations = []
        if hours_score > 0.7:
            recommendations.append("Reduce daily working hours")
        if task_score > 0.7:
            recommendations.append("Redistribute tasks to team members")
        if duration_score > 0.7:
            recommendations.append("Schedule mandatory time off")
        if completion_score > 0.6:
            recommendations.append("Review task complexity and deadlines")
        
        return {
            "burnoutRisk": float(burnout_risk),
            "recommendations": recommendations
        }
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Dict, Optional
from models.risk_predictor import RiskPredictor
from models.burnout_analyzer import BurnoutAnalyzer
import os

app = FastAPI(title="Enterprise ML Service")

risk_predictor = RiskPredictor()
burnout_analyzer = BurnoutAnalyzer()

class RiskFeatures(BaseModel):
    userId: str
    features: Dict[str, float]

class BurnoutData(BaseModel):
    userId: str
    workloadData: Dict[str, float]

class ProjectData(BaseModel):
    projectId: str
    features: Dict[str, float]

@app.post("/ml/predict-risk")
async def predict_risk(data: RiskFeatures):
    try:
        result = risk_predictor.predict(data.features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/analyze-burnout")
async def analyze_burnout(data: BurnoutData):
    try:
        result = burnout_analyzer.analyze(data.workloadData)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/predict-delay")
async def predict_delay(data: ProjectData):
    try:
        features = data.features
        progress_rate = features['tasksCompleted'] / features['tasksTotal']
        expected_rate = (features['tasksTotal'] - features['daysRemaining'] * features['avgVelocity']) / features['tasksTotal']
        
        delay_probability = max(0, min(1, (expected_rate - progress_rate) * 2))
        
        if delay_probability > 0.5:
            days_needed = (features['tasksTotal'] - features['tasksCompleted']) / features['avgVelocity']
            estimated_delay = max(0, days_needed - features['daysRemaining'])
        else:
            estimated_delay = 0
        
        return {
            "delayProbability": float(delay_probability),
            "estimatedDelay": int(estimated_delay)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/train-risk-model")
async def train_risk_model(features: Dict[str, float], label: int):
    """Endpoint for online learning - admin can submit training data"""
    try:
        risk_predictor.train(features, label)
        return {"status": "Model updated successfully"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
NODE_ENV=development
```

**ML Service (.env)**
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
WORKERS=4
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

### Docker Deployment

```yaml
# docker-compose.yml
version: '3.8'

services:
  mongodb:
    image: mongo:latest
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/enterprise-user-mgmt
      - JWT_SECRET=${JWT_SECRET}
      - ML_SERVICE_URL=http://ml-service:8000
    depends_on:
      - mongodb

  ml-service:
    build: ./ml-service
    ports:
      - "8000:8000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/enterprise-user-mgmt
    volumes:
      - ml-models:/app/models
    depends_on:
      - mongodb

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:5000
      - REACT_APP_ML_API_URL=http://localhost:8000
    depends_on:
      - backend

volumes:
  mongo-data:
  ml-models:
```

## Common Patterns

### Protected Route Component

```javascript
// src/components/ProtectedRoute.js
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const { user, token } = useContext(AuthContext);

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requireAdmin && user?.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Time Tracker Hook

```javascript
// src/hooks/useTimeTracker.js
import { useState, useEffect } from 'react';
import api from '../utils/axios';

export const useTimeTracker = (taskId) => {
  const [isTracking, setIsTracking] = useState(false);
  const [elapsed, setElapsed] = useState(0);
  const [startTime, setStartTime] = useState(null);

  useEffect(() => {
    let interval;
    if (isTracking) {
      interval = setInterval(() => {
        setElapsed(Math.floor((Date.now() - startTime) / 1000));
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isTracking, startTime]);

  const start = () => {
    setIsTracking(true);
    setStartTime(Date.now());
  };

  const stop = async () => {
    setIsTracking(false);
    if (elapsed > 0) {
      await api.post(`/api/tasks/${taskId}/time`, {
        duration: elapsed,
        date: new Date()
      });
    }
    setElapsed(0);
  };

  return { isTracking, elapsed, start, stop };
};
```

### Notification System

```javascript
// src/utils/notifications.js
import api from './axios';

export const subscribeToNotifications = (userId, callback) => {
  const eventSource = new EventSource(
    `${process.env.REACT_APP_API_URL}/api/notifications/stream?userId=${userId}`
  );

  eventSource.onmessage = (event) => {
    const notification = JSON.parse(event.data);
    callback(notification);
  };

  return () => eventSource.close();
};

export const markAsRead = async (notificationId) => {
  await api.patch(`/api/notifications/${notificationId}/read`);
};
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check MongoDB status
sudo systemctl status mongodb

# Restart MongoDB
sudo systemctl restart mongodb

# Check connection string
echo $MONGODB_URI
```

### JWT Token Expiration

```javascript
// backend/controllers/authController.js
exports.refreshToken = async (req, res) => {
  const { refreshToken } = req.body;
  
  try {
    const decoded = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET);
    const user = await User.findById(decoded.id);
    
    
