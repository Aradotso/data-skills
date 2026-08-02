---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "configure user task tracking with AI insights"
  - "implement risk detection and anomaly detection"
  - "build admin dashboard with user management"
  - "create AI-powered ticket classification system"
  - "add burnout detection and predictive analytics"
  - "deploy enterprise user management with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack web application for managing users, tasks, and support tickets with integrated AI analytics for risk detection, anomaly detection, burnout analysis, and predictive insights.

## What It Does

This system provides:
- **User Management**: JWT-based authentication, role-based access control
- **Task Tracking**: Kanban boards, time tracking, progress monitoring
- **Support Tickets**: Smart ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, user monitoring

## Installation

### Prerequisites
- Node.js 14+
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
MONGO_URI=mongodb://localhost:27017/enterprise_users
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
MODEL_PATH=./models
DB_URI=mongodb://localhost:27017/enterprise_users
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
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Key API Endpoints

### Authentication (Backend)

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
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
const loginUser = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Backend)

```javascript
// GET /api/users - Get all users (Admin only)
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id - Delete user (Admin only)
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management (Backend)

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo,
      priority: taskData.priority,
      dueDate: taskData.dueDate,
      status: 'todo'
    })
  });
  return response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};
```

### Support Tickets (Backend)

```javascript
// POST /api/tickets - Create ticket
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category,
      priority: ticketData.priority
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets
const getAllTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## AI/ML Service API

### Risk Detection

```javascript
// POST /api/ml/risk-detection
const detectRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.userId,
      loginFrequency: userData.loginFrequency,
      taskCompletionRate: userData.taskCompletionRate,
      failedLogins: userData.failedLogins,
      anomalyScore: userData.anomalyScore
    })
  });
  return response.json();
  // Returns: { riskScore: 0.75, riskLevel: "high", factors: [...] }
};
```

### Anomaly Detection

```javascript
// POST /api/ml/anomaly-detection
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: activityData.userId,
      timestamp: activityData.timestamp,
      activity: activityData.activity,
      location: activityData.location,
      deviceInfo: activityData.deviceInfo
    })
  });
  return response.json();
  // Returns: { isAnomaly: true, score: 0.89, reason: "unusual login location" }
};
```

### Burnout Detection

```javascript
// POST /api/ml/burnout-detection
const detectBurnout = async (workloadData) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: workloadData.userId,
      tasksCompleted: workloadData.tasksCompleted,
      averageTaskTime: workloadData.averageTaskTime,
      overtimeHours: workloadData.overtimeHours,
      missedDeadlines: workloadData.missedDeadlines,
      weeklyWorkload: workloadData.weeklyWorkload
    })
  });
  return response.json();
  // Returns: { burnoutRisk: "moderate", score: 0.62, recommendations: [...] }
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
      title: ticketText.title,
      description: ticketText.description
    })
  });
  return response.json();
  // Returns: { category: "technical", priority: "high", suggestedAssignee: "tech-team" }
};
```

### Predictive Project Insights

```javascript
// POST /api/ml/predict-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.projectId,
      tasksRemaining: projectData.tasksRemaining,
      teamSize: projectData.teamSize,
      completionRate: projectData.completionRate,
      daysToDeadline: projectData.daysToDeadline
    })
  });
  return response.json();
  // Returns: { delayProbability: 0.73, estimatedDelay: 5, recommendations: [...] }
};
```

## React Component Examples

### User Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [burnoutScore, setBurnoutScore] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUserData();
  }, []);

  const fetchUserData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      // Fetch tasks
      const tasksRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks/user/me`,
        config
      );
      setTasks(tasksRes.data);

      // Fetch burnout analysis
      const burnoutRes = await axios.post(
        `${process.env.REACT_APP_ML_URL}/api/ml/burnout-detection`,
        {
          userId: localStorage.getItem('userId'),
          tasksCompleted: tasksRes.data.filter(t => t.status === 'done').length,
          weeklyWorkload: tasksRes.data.length
        }
      );
      setBurnoutScore(burnoutRes.data);
      
      setLoading(false);
    } catch (error) {
      console.error('Error fetching data:', error);
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchUserData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {burnoutScore && burnoutScore.burnoutRisk !== 'low' && (
        <div className="alert-warning">
          ⚠️ Burnout Risk: {burnoutScore.burnoutRisk}
        </div>
      )}

      <div className="kanban-board">
        {['todo', 'inprogress', 'done'].map(status => (
          <div key={status} className="kanban-column">
            <h3>{status.toUpperCase()}</h3>
            {tasks
              .filter(task => task.status === status)
              .map(task => (
                <div key={task._id} className="task-card">
                  <h4>{task.title}</h4>
                  <p>{task.description}</p>
                  <button onClick={() => updateTaskStatus(task._id, 
                    status === 'todo' ? 'inprogress' : 'done')}>
                    Move →
                  </button>
                </div>
              ))}
          </div>
        ))}
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [users, setUsers] = useState([]);
  const [riskUsers, setRiskUsers] = useState([]);
  const [analytics, setAnalytics] = useState({});

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = { headers: { Authorization: `Bearer ${token}` } };

      // Fetch all users
      const usersRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/users`,
        config
      );
      setUsers(usersRes.data);

      // Analyze risk for each user
      const riskAnalysis = await Promise.all(
        usersRes.data.map(async (user) => {
          const risk = await axios.post(
            `${process.env.REACT_APP_ML_URL}/api/ml/risk-detection`,
            {
              userId: user._id,
              loginFrequency: user.loginCount || 0,
              taskCompletionRate: user.completionRate || 0,
              failedLogins: user.failedLogins || 0
            }
          );
          return { ...user, risk: risk.data };
        })
      );

      const highRisk = riskAnalysis.filter(u => u.risk.riskLevel === 'high');
      setRiskUsers(highRisk);

      // Get analytics summary
      const analyticsRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/analytics/summary`,
        config
      );
      setAnalytics(analyticsRes.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="admin-analytics">
      <h1>Admin Analytics Dashboard</h1>

      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-number">{users.length}</p>
        </div>
        <div className="stat-card">
          <h3>High Risk Users</h3>
          <p className="stat-number red">{riskUsers.length}</p>
        </div>
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p className="stat-number">{analytics.activeTasks || 0}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p className="stat-number">{analytics.openTickets || 0}</p>
        </div>
      </div>

      {riskUsers.length > 0 && (
        <div className="risk-alerts">
          <h2>⚠️ High Risk Users</h2>
          <table>
            <thead>
              <tr>
                <th>Name</th>
                <th>Email</th>
                <th>Risk Score</th>
                <th>Factors</th>
              </tr>
            </thead>
            <tbody>
              {riskUsers.map(user => (
                <tr key={user._id}>
                  <td>{user.name}</td>
                  <td>{user.email}</td>
                  <td>{(user.risk.riskScore * 100).toFixed(0)}%</td>
                  <td>{user.risk.factors?.join(', ')}</td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      )}
    </div>
  );
};

export default AdminAnalytics;
```

## Backend API Implementation Pattern

### Express.js Route Structure

```javascript
// routes/tasks.js
const express = require('express');
const router = express.Router();
const auth = require('../middleware/auth');
const Task = require('../models/Task');
const axios = require('axios');

// Create task
router.post('/', auth, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    await task.save();

    // Predict potential delay
    const prediction = await axios.post(
      `${process.env.ML_SERVICE_URL}/api/ml/predict-delay`,
      {
        projectId: task.projectId,
        tasksRemaining: await Task.countDocuments({ 
          projectId: task.projectId, 
          status: { $ne: 'done' } 
        }),
        daysToDeadline: Math.ceil(
          (new Date(task.dueDate) - new Date()) / (1000 * 60 * 60 * 24)
        )
      }
    );

    res.status(201).json({ task, prediction: prediction.data });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tasks
router.get('/user/:userId', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { 
        status: req.body.status,
        updatedAt: Date.now()
      },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### JWT Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

module.exports = function(req, res, next) {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No token, authorization denied' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded.user;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Token is not valid' });
  }
};
```

## ML Service Implementation (Python/FastAPI)

### Risk Detection Model

```python
# ml-service/models/risk_detection.py
from river import linear_model, preprocessing
import pickle
import os

class RiskDetectionModel:
    def __init__(self):
        self.model = linear_model.LogisticRegression()
        self.scaler = preprocessing.StandardScaler()
        self.load_model()
    
    def load_model(self):
        model_path = os.getenv('MODEL_PATH', './models')
        try:
            with open(f'{model_path}/risk_model.pkl', 'rb') as f:
                self.model = pickle.load(f)
        except FileNotFoundError:
            pass  # Use fresh model
    
    def predict_risk(self, features):
        # features: dict with loginFrequency, taskCompletionRate, etc.
        scaled_features = {}
        for key, value in features.items():
            scaled_features[key] = self.scaler.learn_one({key: value})[key]
        
        risk_score = self.model.predict_proba_one(scaled_features).get(1, 0)
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = "high"
        elif risk_score >= 0.4:
            risk_level = "moderate"
        else:
            risk_level = "low"
        
        # Identify risk factors
        factors = []
        if features.get('failedLogins', 0) > 3:
            factors.append("high failed login attempts")
        if features.get('taskCompletionRate', 1) < 0.5:
            factors.append("low task completion rate")
        if features.get('loginFrequency', 0) < 2:
            factors.append("low activity")
        
        return {
            "riskScore": float(risk_score),
            "riskLevel": risk_level,
            "factors": factors
        }
    
    def train_online(self, features, label):
        """Online learning - update model with new data"""
        self.model.learn_one(features, label)
```

### FastAPI Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.risk_detection import RiskDetectionModel
from models.anomaly_detection import AnomalyDetector
from models.burnout_detection import BurnoutDetector
from models.ticket_classifier import TicketClassifier

app = FastAPI()

# Initialize models
risk_model = RiskDetectionModel()
anomaly_detector = AnomalyDetector()
burnout_detector = BurnoutDetector()
ticket_classifier = TicketClassifier()

class RiskDetectionRequest(BaseModel):
    userId: str
    loginFrequency: int
    taskCompletionRate: float
    failedLogins: int
    anomalyScore: float = 0.0

class BurnoutDetectionRequest(BaseModel):
    userId: str
    tasksCompleted: int
    averageTaskTime: float = 0
    overtimeHours: float = 0
    missedDeadlines: int = 0
    weeklyWorkload: int

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

@app.post("/api/ml/risk-detection")
async def detect_risk(request: RiskDetectionRequest):
    try:
        features = {
            "loginFrequency": request.loginFrequency,
            "taskCompletionRate": request.taskCompletionRate,
            "failedLogins": request.failedLogins,
            "anomalyScore": request.anomalyScore
        }
        result = risk_model.predict_risk(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        features = {
            "tasksCompleted": request.tasksCompleted,
            "averageTaskTime": request.averageTaskTime,
            "overtimeHours": request.overtimeHours,
            "missedDeadlines": request.missedDeadlines,
            "weeklyWorkload": request.weeklyWorkload
        }
        result = burnout_detector.analyze(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        result = ticket_classifier.classify(
            request.title,
            request.description
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "models_loaded": True}
```

## Configuration

### MongoDB Schema Examples

```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  loginCount: { type: Number, default: 0 },
  failedLogins: { type: Number, default: 0 },
  lastLogin: { type: Date },
  completionRate: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', userSchema);
```

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['todo', 'inprogress', 'done'], 
    default: 'todo' 
  },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  projectId: String,
  timeTracked: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

## Common Patterns

### Protected Route Pattern

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const userRole = localStorage.getItem('role');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && userRole !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Axios Interceptor for Auth

```javascript
// utils/axios.js
import axios from 'axios';

const axiosInstance = axios.create({
  baseURL: process.env.REACT_APP_API_URL
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

### JWT Token Expired
```javascript
// Check token expiration before making requests
const isTokenExpired = () => {
  const token = localStorage.getItem('token');
  if (!token) return true;
  
  try {
    const payload = JSON.parse(atob(token.split('.')[1]));
    return payload.exp * 1000 < Date.now();
  } catch (e) {
    return true;
  }
};

if (isTokenExpired()) {
  // Redirect to login
  window.location.href = '/login';
}
```

### CORS Issues
Backend configuration:
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Connection Error
```javascript
// Check ML service health before making predictions
const checkMLService = async () => {
  try {
    const response = await fetch(`${process.env.REACT_APP_ML_URL}/health`);
    return response.ok;
  } catch (error) {
    console.error('ML service unavailable:', error);
    return false;
  }
};
```

### MongoDB Connection Issues
```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error.message);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Model Not Training
For online learning models:
```python
# Ensure model persistence
def save_model(model, path):
    import pickle
    with open(path, 'wb') as f
