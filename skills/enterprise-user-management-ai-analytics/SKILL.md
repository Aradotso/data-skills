---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management capabilities
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics for user management"
  - "set up user management with task tracking and AI insights"
  - "configure the enterprise user management API"
  - "implement AI-based risk detection for users"
  - "create a user dashboard with task management"
  - "how do I use the ML service for anomaly detection"
  - "build an admin panel with user management features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user management, task tracking, and support ticket handling with AI-powered insights. It provides risk detection, anomaly detection, burnout analysis, and predictive project insights using machine learning models.

The system consists of three main components:
- **Frontend**: React.js application for admin and user dashboards
- **Backend**: Node.js/Express REST API with MongoDB
- **ML Service**: FastAPI-based microservice with scikit-learn and River for online learning

## Installation

### Prerequisites

```bash
# Required
node --version  # v14+
python --version  # 3.8+
mongodb --version  # 4.4+
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:
```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
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
# Frontend runs at http://localhost:3000
```

## Backend API Usage

### Authentication

**Register User**
```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user' // 'admin' or 'user'
    })
  });
  return response.json();
};
```

**Login**
```javascript
// POST /api/auth/login
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  const data = await response.json();
  // Store token for subsequent requests
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

**Get All Users**
```javascript
// GET /api/users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};
```

**Update User**
```javascript
// PUT /api/users/:id
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
```

**Delete User**
```javascript
// DELETE /api/users/:id
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management

**Create Task**
```javascript
// POST /api/tasks
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
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'in-progress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};
```

**Get User Tasks**
```javascript
// GET /api/tasks/user/:userId
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

**Update Task Status**
```javascript
// PATCH /api/tasks/:id/status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'todo', 'in-progress', 'done'
  });
  return response.json();
};
```

**Track Time**
```javascript
// POST /api/tasks/:id/time-log
const logTaskTime = async (taskId, timeData, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time-log`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      startTime: timeData.startTime,
      endTime: timeData.endTime,
      duration: timeData.duration // in seconds
    })
  });
  return response.json();
};
```

### Support Tickets

**Create Ticket**
```javascript
// POST /api/tickets
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
      priority: ticketData.priority, // 'low', 'medium', 'high', 'critical'
      category: ticketData.category // 'technical', 'billing', 'general'
    })
  });
  return response.json();
};
```

**Get All Tickets (Admin)**
```javascript
// GET /api/tickets
const getAllTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

## ML Service API Usage

### Risk Detection

```javascript
// POST /api/ml/risk-detection
const analyzeUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      features: {
        login_failures: 3,
        access_frequency: 150,
        unusual_hours: 5,
        data_access_volume: 2000
      }
    })
  });
  const data = await response.json();
  // Returns: { risk_score: 0.75, risk_level: 'high', factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/anomaly-detection
const detectAnomaly = async (behaviorData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: behaviorData.userId,
      timestamp: new Date().toISOString(),
      features: {
        session_duration: behaviorData.sessionDuration,
        actions_per_minute: behaviorData.actionsPerMinute,
        resource_access_count: behaviorData.resourceAccessCount,
        location_change: behaviorData.locationChange
      }
    })
  });
  const data = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.92, explanation: '...' }
  return data;
};
```

### Burnout Detection

```javascript
// POST /api/ml/burnout-detection
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      metrics: {
        tasks_completed: 45,
        overtime_hours: 20,
        missed_deadlines: 3,
        average_task_completion_time: 180, // minutes
        workload_score: 8.5
      }
    })
  });
  const data = await response.json();
  // Returns: { burnout_risk: 'high', score: 0.82, recommendations: [...] }
  return data;
};
```

### Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      subject: ticketText.subject,
      description: ticketText.description
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.89 }
  return data;
};
```

### Project Delay Prediction

```javascript
// POST /api/ml/predict-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.id,
      features: {
        total_tasks: projectData.totalTasks,
        completed_tasks: projectData.completedTasks,
        team_size: projectData.teamSize,
        days_remaining: projectData.daysRemaining,
        average_velocity: projectData.avgVelocity
      }
    })
  });
  const data = await response.json();
  // Returns: { delay_probability: 0.65, estimated_delay_days: 5, risk_factors: [...] }
  return data;
};
```

## Frontend Component Patterns

### Protected Route with Authentication

```javascript
// components/ProtectedRoute.js
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('token');
  const userRole = localStorage.getItem('role');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (adminOnly && userRole !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### Task Kanban Board Component

```javascript
// components/TaskBoard.js
import React, { useState, useEffect } from 'react';

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/tasks/user/${userId}`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    });
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await fetch(
      `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
      {
        method: 'PATCH',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ status: newStatus })
      }
    );
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="kanban-column">
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

### AI Risk Dashboard Component

```javascript
// components/RiskDashboard.js
import React, { useState, useEffect } from 'react';

const RiskDashboard = () => {
  const [riskData, setRiskData] = useState([]);
  const token = localStorage.getItem('token');

  useEffect(() => {
    analyzeAllUsers();
  }, []);

  const analyzeAllUsers = async () => {
    const usersResponse = await fetch(
      `${process.env.REACT_APP_API_URL}/users`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const users = await usersResponse.json();

    const riskAnalyses = await Promise.all(
      users.map(async (user) => {
        const riskResponse = await fetch(
          `${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/risk-detection`,
          {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              user_id: user._id,
              features: user.securityMetrics || {}
            })
          }
        );
        const risk = await riskResponse.json();
        return { ...user, risk };
      })
    );

    setRiskData(riskAnalyses.filter(u => u.risk.risk_level !== 'low'));
  };

  return (
    <div className="risk-dashboard">
      <h2>High Risk Users</h2>
      {riskData.map(user => (
        <div key={user._id} className={`risk-card ${user.risk.risk_level}`}>
          <h3>{user.name}</h3>
          <p>Risk Score: {(user.risk.risk_score * 100).toFixed(1)}%</p>
          <p>Level: {user.risk.risk_level}</p>
          <ul>
            {user.risk.factors?.map((factor, idx) => (
              <li key={idx}>{factor}</li>
            ))}
          </ul>
        </div>
      ))}
    </div>
  );
};

export default RiskDashboard;
```

## Configuration

### Backend Configuration Options

```javascript
// backend/config/config.js
module.exports = {
  port: process.env.PORT || 5000,
  mongoUri: process.env.MONGODB_URI,
  jwtSecret: process.env.JWT_SECRET,
  jwtExpire: process.env.JWT_EXPIRE || '7d',
  mlServiceUrl: process.env.ML_SERVICE_URL || 'http://localhost:8000',
  
  // Security settings
  maxLoginAttempts: 5,
  lockoutDuration: 15 * 60 * 1000, // 15 minutes
  
  // Task settings
  taskStatuses: ['todo', 'in-progress', 'done'],
  taskPriorities: ['low', 'medium', 'high'],
  
  // Ticket settings
  ticketCategories: ['technical', 'billing', 'general', 'feature-request'],
  ticketPriorities: ['low', 'medium', 'high', 'critical']
};
```

### ML Service Configuration

```python
# ml-service/config.py
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    model_path: str = os.getenv('MODEL_PATH', './models')
    log_level: str = os.getenv('LOG_LEVEL', 'INFO')
    backend_url: str = os.getenv('BACKEND_URL', 'http://localhost:5000')
    
    # Model thresholds
    risk_threshold_high: float = 0.7
    risk_threshold_medium: float = 0.4
    anomaly_threshold: float = 0.8
    burnout_threshold_high: float = 0.75
    
    # Feature importance weights
    risk_weights: dict = {
        'login_failures': 0.3,
        'unusual_hours': 0.25,
        'data_access_volume': 0.25,
        'access_frequency': 0.2
    }
    
    class Config:
        env_file = '.env'

settings = Settings()
```

## Common Patterns

### Middleware for JWT Authentication

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
  } catch (error) {
    res.status(401).json({ message: 'Token invalid' });
  }
};

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: 'Not authorized to access this route' 
      });
    }
    next();
  };
};

module.exports = { protect, authorize };
```

### Database Models

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'User', 
    required: true 
  },
  createdBy: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'User' 
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
  dueDate: { type: Date },
  timeLogs: [{
    startTime: Date,
    endTime: Date,
    duration: Number // in seconds
  }],
  tags: [String]
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

### ML Model Training Script

```python
# ml-service/models/risk_detector.py
from sklearn.ensemble import RandomForestClassifier
import joblib
import numpy as np

class RiskDetector:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        try:
            self.model = joblib.load(model_path)
        except:
            self.model = RandomForestClassifier(n_estimators=100, random_state=42)
            self.is_trained = False
    
    def extract_features(self, user_data):
        """Extract features from user behavior data"""
        features = [
            user_data.get('login_failures', 0),
            user_data.get('access_frequency', 0),
            user_data.get('unusual_hours', 0),
            user_data.get('data_access_volume', 0)
        ]
        return np.array(features).reshape(1, -1)
    
    def predict_risk(self, user_data):
        """Predict risk score for a user"""
        features = self.extract_features(user_data)
        
        if hasattr(self.model, 'predict_proba'):
            risk_score = self.model.predict_proba(features)[0][1]
        else:
            risk_score = self.model.predict(features)[0]
        
        risk_level = 'high' if risk_score > 0.7 else 'medium' if risk_score > 0.4 else 'low'
        
        return {
            'risk_score': float(risk_score),
            'risk_level': risk_level,
            'factors': self._identify_risk_factors(user_data, risk_score)
        }
    
    def _identify_risk_factors(self, user_data, risk_score):
        """Identify specific risk factors"""
        factors = []
        if user_data.get('login_failures', 0) > 3:
            factors.append('Multiple failed login attempts')
        if user_data.get('unusual_hours', 0) > 5:
            factors.append('Access during unusual hours')
        if user_data.get('data_access_volume', 0) > 1000:
            factors.append('High data access volume')
        return factors
    
    def save_model(self):
        """Save trained model to disk"""
        joblib.dump(self.model, self.model_path)
```

## Troubleshooting

### Common Issues

**Issue: JWT Token Expired**
```javascript
// Solution: Implement token refresh mechanism
const refreshToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  const response = await fetch(`${API_URL}/auth/refresh`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken })
  });
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data.token;
};
```

**Issue: ML Service Connection Timeout**
```javascript
// Solution: Add retry logic with exponential backoff
const callMLService = async (endpoint, data, retries = 3) => {
  for (let i = 0; i < retries; i++) {
    try {
      const response = await fetch(`${ML_SERVICE_URL}${endpoint}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
        timeout: 5000
      });
      return await response.json();
    } catch (error) {
      if (i === retries - 1) throw error;
      await new Promise(r => setTimeout(r, 1000 * Math.pow(2, i)));
    }
  }
};
```

**Issue: MongoDB Connection Fails**
```javascript
// Solution: Add connection retry with proper error handling
const connectDB = async () => {
  const options = {
    useNewUrlParser: true,
    useUnifiedTopology: true,
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 45000,
  };
  
  try {
    await mongoose.connect(process.env.MONGODB_URI, options);
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    setTimeout(connectDB, 5000); // Retry after 5 seconds
  }
};
```

**Issue: CORS Errors in Development**
```javascript
// backend/app.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

**Issue: ML Model Not Loading**
```python
# ml-service/main.py
import os
from pathlib import Path

# Ensure model directory exists
model_dir = Path(os.getenv('MODEL_PATH', './models'))
model_dir.mkdir(parents=True, exist_ok=True)

# Initialize models with fallback
try:
    risk_detector = RiskDetector(str(model_dir / 'risk_model.pkl'))
except Exception as e:
    print(f"Warning: Could not load risk model: {e}")
    risk_detector = RiskDetector()  # Use untrained model
```

## Performance Optimization

### Caching User Data

```javascript
// backend/middleware/cache.js
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 600 }); // 10 minutes

const cacheMiddleware = (duration) => (req, res, next) => {
  const key = `__express__${req.originalUrl}` || req.url;
  const cachedBody = cache.get(key);
  
  if (cachedBody) {
    return res.send(cachedBody);
  }
  
  res.sendResponse = res.send;
  res.send = (body) => {
    cache.set(key, body, duration);
    res.sendResponse(body);
  };
  next();
};

module.exports = cacheMiddleware;
```

### Batch ML Predictions

```python
# ml-service/utils/batch_processor.py
async def batch_predict_risks(user_ids: list):
    """Process multiple risk predictions in batch"""
    risk_detector = RiskDetector()
    
    # Fetch all user data in parallel
    user_data_list = await asyncio.gather(*[
        get_user_metrics(user_id) for user_id in user_ids
    ])
    
    # Batch predict
    predictions = [
        risk_detector.predict_risk(data) 
        for data in user_data_list
    ]
    
    return dict(zip(user_ids, predictions))
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI coding agents to effectively assist developers in implementing, extending, and troubleshooting the system.
