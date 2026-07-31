---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered risk detection, anomaly analysis, and predictive insights built with React, Node.js, and FastAPI
triggers:
  - "set up enterprise user management system with AI"
  - "integrate AI analytics for user management"
  - "build user management dashboard with risk detection"
  - "implement ticket classification and burnout analysis"
  - "create admin dashboard with AI insights"
  - "deploy enterprise user management with ML service"
  - "configure JWT authentication for user management"
  - "add anomaly detection to user management system"
---

# Enterprise User Management AI Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System with AI Analytics - a full-stack application that combines traditional user/task management with intelligent features like risk prediction, anomaly detection, burnout analysis, and automated ticket classification.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk detection, anomaly detection, burnout prediction, project delay forecasting
- **Dashboards**: Admin analytics and user performance insights

## Architecture

The system consists of three main components:
- **Frontend** (React.js): User/admin dashboards, Kanban boards
- **Backend** (Node.js): REST APIs, authentication, business logic
- **ML Service** (FastAPI): AI/ML models for predictions and analytics

## Installation

### Prerequisites

Ensure you have installed:
- Node.js (v14+)
- Python (3.8+)
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

Create `.env` file in `backend/` directory:

```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
```

Start the backend:

```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in `ml-service/` directory:

```env
MODEL_PATH=./models
DB_URI=mongodb://localhost:27017/enterprise-ums
LOG_LEVEL=INFO
```

Start the ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in `frontend/` directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start the frontend:

```bash
npm start
```

## Backend API Structure

### Authentication Endpoints

**User Login**:
```javascript
// POST /api/auth/login
const response = await fetch('http://localhost:5000/api/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'user@example.com',
    password: 'password123'
  })
});
const { token, user } = await response.json();
```

**User Registration**:
```javascript
// POST /api/auth/register
const response = await fetch('http://localhost:5000/api/auth/register', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    name: 'John Doe',
    email: 'john@example.com',
    password: 'securepass',
    role: 'user'
  })
});
```

### User Management Endpoints

**Get All Users (Admin)**:
```javascript
// GET /api/users
const response = await fetch('http://localhost:5000/api/users', {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});
const users = await response.json();
```

**Update User**:
```javascript
// PUT /api/users/:id
const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
  method: 'PUT',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: 'Updated Name',
    role: 'admin',
    status: 'active'
  })
});
```

**Delete User**:
```javascript
// DELETE /api/users/:id
await fetch(`http://localhost:5000/api/users/${userId}`, {
  method: 'DELETE',
  headers: { 'Authorization': `Bearer ${token}` }
});
```

### Task Management Endpoints

**Create Task**:
```javascript
// POST /api/tasks
const response = await fetch('http://localhost:5000/api/tasks', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    title: 'Implement feature X',
    description: 'Add new feature to dashboard',
    assignedTo: userId,
    priority: 'high',
    dueDate: '2026-05-01',
    status: 'todo'
  })
});
```

**Update Task Status**:
```javascript
// PATCH /api/tasks/:id/status
await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ status: 'in-progress' })
});
```

**Get User Tasks**:
```javascript
// GET /api/tasks/user/:userId
const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
  headers: { 'Authorization': `Bearer ${token}` }
});
const tasks = await response.json();
```

### Support Ticket Endpoints

**Create Ticket**:
```javascript
// POST /api/tickets
const response = await fetch('http://localhost:5000/api/tickets', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    title: 'Login Issue',
    description: 'Cannot access dashboard after password reset',
    category: 'technical',
    priority: 'medium'
  })
});
```

**Get All Tickets (Admin)**:
```javascript
// GET /api/tickets
const response = await fetch('http://localhost:5000/api/tickets', {
  headers: { 'Authorization': `Bearer ${token}` }
});
const tickets = await response.json();
```

## ML Service API

### Risk Detection

**Predict User Risk**:
```javascript
// POST /api/ml/predict-risk
const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: userId,
    loginFrequency: 15,
    failedLoginAttempts: 3,
    taskCompletionRate: 0.75,
    avgResponseTime: 120,
    accessPatterns: ['normal', 'normal', 'unusual']
  })
});
const { riskScore, riskLevel, factors } = await response.json();
```

### Anomaly Detection

**Detect Anomalies**:
```javascript
// POST /api/ml/detect-anomaly
const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: userId,
    activityData: {
      loginTime: '2026-04-15T03:30:00Z',
      location: '192.168.1.100',
      actionsPerformed: ['delete_user', 'bulk_export'],
      sessionDuration: 5
    }
  })
});
const { isAnomaly, anomalyScore, reason } = await response.json();
```

### Burnout Analysis

**Analyze Employee Burnout**:
```javascript
// POST /api/ml/burnout-analysis
const response = await fetch('http://localhost:8000/api/ml/burnout-analysis', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: userId,
    workloadData: {
      tasksAssigned: 25,
      tasksCompleted: 18,
      avgHoursPerDay: 10.5,
      weekendWork: true,
      missedDeadlines: 4
    }
  })
});
const { burnoutRisk, score, recommendations } = await response.json();
```

### Ticket Classification

**Classify Support Ticket**:
```javascript
// POST /api/ml/classify-ticket
const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    title: 'Cannot access reports',
    description: 'Getting 403 error when trying to view analytics dashboard',
    content: 'Error occurs after latest update'
  })
});
const { category, priority, assignedTo, confidence } = await response.json();
```

### Project Insights

**Predict Project Delays**:
```javascript
// POST /api/ml/predict-delay
const response = await fetch('http://localhost:8000/api/ml/predict-delay', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    projectId: projectId,
    tasksTotal: 50,
    tasksCompleted: 20,
    daysElapsed: 30,
    daysRemaining: 20,
    teamSize: 5,
    blockers: 3
  })
});
const { delayPredicted, estimatedDelay, suggestions } = await response.json();
```

## Frontend Components

### React Hook for API Calls

```javascript
// hooks/useAPI.js
import { useState, useEffect } from 'react';

export const useAPI = (url, token) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        const response = await fetch(url, {
          headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'
          }
        });
        const result = await response.json();
        setData(result);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, [url, token]);

  return { data, loading, error };
};
```

### User Dashboard Component

```javascript
// components/UserDashboard.jsx
import React from 'react';
import { useAPI } from '../hooks/useAPI';

const UserDashboard = ({ token, userId }) => {
  const { data: tasks, loading } = useAPI(
    `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
    token
  );

  if (loading) return <div>Loading...</div>;

  const todoTasks = tasks?.filter(t => t.status === 'todo') || [];
  const inProgressTasks = tasks?.filter(t => t.status === 'in-progress') || [];
  const doneTasks = tasks?.filter(t => t.status === 'done') || [];

  return (
    <div className="dashboard">
      <h1>My Tasks</h1>
      <div className="kanban-board">
        <TaskColumn title="To Do" tasks={todoTasks} />
        <TaskColumn title="In Progress" tasks={inProgressTasks} />
        <TaskColumn title="Done" tasks={doneTasks} />
      </div>
    </div>
  );
};

const TaskColumn = ({ title, tasks }) => (
  <div className="kanban-column">
    <h2>{title} ({tasks.length})</h2>
    {tasks.map(task => (
      <TaskCard key={task._id} task={task} />
    ))}
  </div>
);

const TaskCard = ({ task }) => (
  <div className="task-card">
    <h3>{task.title}</h3>
    <p>{task.description}</p>
    <span className={`priority-${task.priority}`}>{task.priority}</span>
  </div>
);

export default UserDashboard;
```

### Admin Analytics Component

```javascript
// components/AdminAnalytics.jsx
import React, { useState, useEffect } from 'react';

const AdminAnalytics = ({ token }) => {
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    const fetchAnalytics = async () => {
      const [users, tasks, tickets, risks] = await Promise.all([
        fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
          headers: { 'Authorization': `Bearer ${token}` }
        }).then(r => r.json()),
        fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
          headers: { 'Authorization': `Bearer ${token}` }
        }).then(r => r.json()),
        fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
          headers: { 'Authorization': `Bearer ${token}` }
        }).then(r => r.json()),
        fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/risk-overview`, {
          headers: { 'Authorization': `Bearer ${token}` }
        }).then(r => r.json())
      ]);

      setAnalytics({ users, tasks, tickets, risks });
    };

    fetchAnalytics();
  }, [token]);

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="admin-analytics">
      <div className="stats-grid">
        <StatCard title="Total Users" value={analytics.users.length} />
        <StatCard title="Active Tasks" value={analytics.tasks.filter(t => t.status !== 'done').length} />
        <StatCard title="Open Tickets" value={analytics.tickets.filter(t => t.status === 'open').length} />
        <StatCard title="High Risk Users" value={analytics.risks.highRisk || 0} />
      </div>
    </div>
  );
};

const StatCard = ({ title, value }) => (
  <div className="stat-card">
    <h3>{title}</h3>
    <p className="stat-value">{value}</p>
  </div>
);

export default AdminAnalytics;
```

## Common Patterns

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ message: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Role-Based Access Control

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const { authMiddleware, adminOnly } = require('../middleware/auth');

// All users can get their own profile
router.get('/profile', authMiddleware, async (req, res) => {
  const user = await User.findById(req.user.id).select('-password');
  res.json(user);
});

// Only admins can list all users
router.get('/', authMiddleware, adminOnly, async (req, res) => {
  const users = await User.find().select('-password');
  res.json(users);
});

// Only admins can delete users
router.delete('/:id', authMiddleware, adminOnly, async (req, res) => {
  await User.findByIdAndDelete(req.params.id);
  res.json({ message: 'User deleted' });
});

module.exports = router;
```

### ML Model Integration Pattern

```python
# ml-service/services/risk_predictor.py
from river import ensemble, preprocessing
import joblib
import os

class RiskPredictor:
    def __init__(self):
        model_path = os.getenv('MODEL_PATH', './models')
        try:
            self.model = joblib.load(f'{model_path}/risk_model.pkl')
        except:
            # Initialize new online learning model
            self.model = ensemble.AdaptiveRandomForestClassifier(
                n_models=10,
                seed=42
            )
        
        self.scaler = preprocessing.StandardScaler()
    
    def predict(self, features):
        """
        Predict risk level for a user
        features: dict with keys like loginFrequency, failedLoginAttempts, etc.
        """
        # Extract and normalize features
        feature_vector = {
            'login_freq': features.get('loginFrequency', 0),
            'failed_logins': features.get('failedLoginAttempts', 0),
            'completion_rate': features.get('taskCompletionRate', 0),
            'response_time': features.get('avgResponseTime', 0)
        }
        
        # Normalize
        normalized = self.scaler.learn_one(feature_vector).transform_one(feature_vector)
        
        # Predict
        risk_score = self.model.predict_proba_one(normalized).get(1, 0)
        
        # Determine risk level
        if risk_score > 0.7:
            risk_level = 'high'
        elif risk_score > 0.4:
            risk_level = 'medium'
        else:
            risk_level = 'low'
        
        return {
            'riskScore': round(risk_score, 3),
            'riskLevel': risk_level,
            'factors': self._identify_factors(feature_vector)
        }
    
    def _identify_factors(self, features):
        factors = []
        if features['failed_logins'] > 5:
            factors.append('High failed login attempts')
        if features['completion_rate'] < 0.5:
            factors.append('Low task completion rate')
        if features['response_time'] > 200:
            factors.append('Slow response time')
        return factors

```

### FastAPI Endpoint Example

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from services.risk_predictor import RiskPredictor
from services.anomaly_detector import AnomalyDetector
from services.burnout_analyzer import BurnoutAnalyzer

app = FastAPI(title="Enterprise UMS ML Service")

risk_predictor = RiskPredictor()
anomaly_detector = AnomalyDetector()
burnout_analyzer = BurnoutAnalyzer()

class RiskRequest(BaseModel):
    userId: str
    loginFrequency: int
    failedLoginAttempts: int
    taskCompletionRate: float
    avgResponseTime: float
    accessPatterns: list

class AnomalyRequest(BaseModel):
    userId: str
    activityData: dict

class BurnoutRequest(BaseModel):
    userId: str
    workloadData: dict

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskRequest):
    try:
        result = risk_predictor.predict(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(request: AnomalyRequest):
    try:
        result = anomaly_detector.detect(request.activityData)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(request: BurnoutRequest):
    try:
        result = burnout_analyzer.analyze(request.workloadData)
        return result
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
MONGO_URI=mongodb://localhost:27017/enterprise-ums

# Authentication
JWT_SECRET=your_secure_jwt_secret_key
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password

# Logging
LOG_LEVEL=info
```

### ML Service Environment Variables

```env
# Model Configuration
MODEL_PATH=./models
RETRAIN_INTERVAL=86400

# Database
DB_URI=mongodb://localhost:27017/enterprise-ums

# API Configuration
API_KEY=your_ml_api_key
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5000

# Logging
LOG_LEVEL=INFO
LOG_FILE=./logs/ml-service.log
```

### Frontend Environment Variables

```env
# API Endpoints
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000

# Feature Flags
REACT_APP_ENABLE_AI_FEATURES=true
REACT_APP_ENABLE_ANALYTICS=true

# Environment
REACT_APP_ENV=development
```

## Troubleshooting

### Backend Issues

**MongoDB Connection Failed**:
```javascript
// Check if MongoDB is running
// Verify MONGO_URI in .env
// Try connection with retry logic:

const mongoose = require('mongoose');

const connectDB = async (retries = 5) => {
  for (let i = 0; i < retries; i++) {
    try {
      await mongoose.connect(process.env.MONGO_URI, {
        useNewUrlParser: true,
        useUnifiedTopology: true
      });
      console.log('MongoDB connected');
      return;
    } catch (err) {
      console.log(`Connection attempt ${i + 1} failed: ${err.message}`);
      await new Promise(res => setTimeout(res, 5000));
    }
  }
  process.exit(1);
};
```

**JWT Token Expired**:
```javascript
// Implement token refresh logic
const refreshToken = async (oldToken) => {
  try {
    const response = await fetch(`${API_URL}/api/auth/refresh`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${oldToken}`,
        'Content-Type': 'application/json'
      }
    });
    const { token } = await response.json();
    localStorage.setItem('token', token);
    return token;
  } catch (error) {
    // Redirect to login
    window.location.href = '/login';
  }
};
```

### ML Service Issues

**Model Not Loading**:
```python
# ml-service/services/base_model.py
import logging
import joblib
import os

logger = logging.getLogger(__name__)

def load_model_safe(model_name, default_model):
    """Safely load model with fallback"""
    model_path = os.path.join(os.getenv('MODEL_PATH', './models'), f'{model_name}.pkl')
    
    try:
        if os.path.exists(model_path):
            model = joblib.load(model_path)
            logger.info(f"Loaded {model_name} from {model_path}")
            return model
        else:
            logger.warning(f"Model file not found, using default for {model_name}")
            return default_model
    except Exception as e:
        logger.error(f"Error loading {model_name}: {str(e)}")
        return default_model
```

**Prediction Errors**:
```python
# Add input validation and error handling
from pydantic import BaseModel, validator

class PredictionInput(BaseModel):
    loginFrequency: int
    failedLoginAttempts: int
    taskCompletionRate: float
    
    @validator('taskCompletionRate')
    def validate_rate(cls, v):
        if not 0 <= v <= 1:
            raise ValueError('taskCompletionRate must be between 0 and 1')
        return v
    
    @validator('failedLoginAttempts')
    def validate_attempts(cls, v):
        if v < 0:
            raise ValueError('failedLoginAttempts cannot be negative')
        return v
```

### Frontend Issues

**CORS Errors**:
```javascript
// backend/server.js
const cors = require('cors');

const corsOptions = {
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));
```

**State Management with Tasks**:
```javascript
// Use React Context for global state
import React, { createContext, useContext, useState } from 'react';

const TaskContext = createContext();

export const TaskProvider = ({ children }) => {
  const [tasks, setTasks] = useState([]);
  
  const updateTaskStatus = async (taskId, newStatus, token) => {
    try {
      await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
        method: 'PATCH',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ status: newStatus })
      });
      
      setTasks(prev => prev.map(t => 
        t._id === taskId ? { ...t, status: newStatus } : t
      ));
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };
  
  return (
    <TaskContext.Provider value={{ tasks, setTasks, updateTaskStatus }}>
      {children}
    </TaskContext.Provider>
  );
};

export const useTasks = () => useContext(TaskContext);
```

## Production Deployment

### Docker Compose Setup

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
    environment:
      MONGO_INITDB_DATABASE: enterprise-ums

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - MONGO_URI=mongodb://mongodb:27017/enterprise-ums
      - JWT_SECRET=${JWT_SECRET}
      - ML_SERVICE_URL=http://ml-service:8000
    depends_on:
      - mongodb

  ml-service:
    build: ./ml-service
    ports:
      - "8000:8000"
    environment:
      - DB_URI=mongodb://mongodb:27017/enterprise-ums
      - MODEL_PATH=/app/models
    volumes:
      - ./ml-service/models:/app/models
    depends_on:
      - mongodb

  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    environment:
      - REACT_APP_API_URL=http://backend:5000
      - REACT_APP_ML_API_URL=http://ml-service:8000

volumes:
  mongo-data:
```

### Environment-Specific Configs

```bash
# Production deployment
docker-compose -f docker-compose.prod.yml up -d

# Development with hot reload
docker-compose -f docker-compose.dev.yml up
```

This skill provides comprehensive guidance for working with the Enterprise User Management System with AI Analytics, covering setup, API usage, common patterns, and troubleshooting for all three tiers of the application.
