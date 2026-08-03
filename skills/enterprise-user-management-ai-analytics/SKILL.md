---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and automated insights for enterprise workflows
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "add AI ticket classification and routing"
  - "build admin panel with user management"
  - "integrate AI risk detection and burnout analysis"
  - "deploy user management with ML service"
  - "configure enterprise analytics dashboard"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines user/task management with AI-powered analytics. It provides role-based access control, Kanban-style task tracking, support ticket management, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and predictive project delays.

**Architecture:**
- **Frontend:** React.js (port 3000)
- **Backend:** Node.js REST API (port 5000)
- **ML Service:** FastAPI with scikit-learn and River (port 8000)
- **Database:** MongoDB
- **Auth:** JWT-based authentication

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB (local or cloud instance)

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

Create `.env` file in `backend/`:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRES_IN=7d
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

Create `.env` file in `ml-service/`:
```env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
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

Create `.env` file in `frontend/`:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Backend API Structure

### Authentication Endpoints

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRES_IN }
    );
    
    res.status(201).json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const isValidPassword = await bcrypt.compare(password, user.password);
    if (!isValidPassword) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRES_IN }
    );
    
    res.json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ message: 'No authentication token' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Task Management Endpoints

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

// Get user tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('assignedBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching tasks', error: error.message });
  }
});

// Create task (admin only)
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, deadline } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.userId,
      priority: priority || 'medium',
      status: 'todo',
      deadline
    });
    
    await task.save();
    await task.populate('assignedTo assignedBy', 'username email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error creating task', error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error updating task', error: error.message });
  }
});

// Track time
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { timeSpent } = req.body; // in minutes
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.timeTracking = task.timeTracking || [];
    task.timeTracking.push({
      date: new Date(),
      minutes: timeSpent,
      userId: req.user.userId
    });
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error tracking time', error: error.message });
  }
});

module.exports = router;
```

### Support Ticket Endpoints

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authMiddleware } = require('../middleware/auth');

// Create support ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    let category = 'general';
    let aiPriority = priority;
    
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        title,
        description
      });
      category = mlResponse.data.category;
      aiPriority = mlResponse.data.priority;
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      createdBy: req.user.userId,
      priority: aiPriority,
      category,
      status: 'open'
    });
    
    await ticket.save();
    await ticket.populate('createdBy', 'username email');
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: 'Error creating ticket', error: error.message });
  }
});

// Get tickets
router.get('/', authMiddleware, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.userId };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching tickets', error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, List
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, linear_model
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
ticket_classifier = None
risk_predictor = None
anomaly_detector = anomaly.HalfSpaceTrees(seed=42)

class TicketRequest(BaseModel):
    title: str
    description: str

class RiskAnalysisRequest(BaseModel):
    user_id: str
    failed_logins: int
    tasks_completed: int
    tasks_overdue: int
    avg_completion_time: float
    access_violations: int

class BurnoutRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_working_hours: float
    missed_deadlines: int
    time_since_break: int  # days

@app.on_event("startup")
async def load_models():
    global ticket_classifier, risk_predictor
    model_path = os.getenv('MODEL_PATH', './models')
    
    # Load or initialize models
    try:
        ticket_classifier = joblib.load(f'{model_path}/ticket_classifier.pkl')
    except:
        ticket_classifier = RandomForestClassifier(n_estimators=100)
    
    try:
        risk_predictor = joblib.load(f'{model_path}/risk_predictor.pkl')
    except:
        risk_predictor = RandomForestClassifier(n_estimators=50)

@app.post("/classify-ticket")
async def classify_ticket(request: TicketRequest):
    """AI-based ticket classification and priority routing"""
    text = f"{request.title} {request.description}".lower()
    
    # Simple rule-based classification (replace with trained model)
    categories = {
        'bug': ['error', 'bug', 'crash', 'broken', 'not working'],
        'feature': ['add', 'feature', 'enhancement', 'improve'],
        'access': ['login', 'password', 'access', 'permission'],
        'performance': ['slow', 'performance', 'timeout', 'lag']
    }
    
    category = 'general'
    for cat, keywords in categories.items():
        if any(keyword in text for keyword in keywords):
            category = cat
            break
    
    # Priority detection
    priority = 'medium'
    if any(word in text for word in ['urgent', 'critical', 'down', 'crash']):
        priority = 'high'
    elif any(word in text for word in ['minor', 'suggestion', 'nice to have']):
        priority = 'low'
    
    return {
        "category": category,
        "priority": priority,
        "confidence": 0.85
    }

@app.post("/analyze-risk")
async def analyze_risk(request: RiskAnalysisRequest):
    """Risk prediction based on user behavior"""
    features = np.array([[
        request.failed_logins,
        request.tasks_completed,
        request.tasks_overdue,
        request.avg_completion_time,
        request.access_violations
    ]])
    
    # Calculate risk score
    risk_score = (
        request.failed_logins * 0.3 +
        request.access_violations * 0.4 +
        request.tasks_overdue * 0.2 +
        max(0, request.avg_completion_time - 48) * 0.1
    )
    
    risk_level = 'low'
    if risk_score > 10:
        risk_level = 'high'
    elif risk_score > 5:
        risk_level = 'medium'
    
    return {
        "user_id": request.user_id,
        "risk_score": float(risk_score),
        "risk_level": risk_level,
        "factors": {
            "failed_logins": request.failed_logins > 3,
            "access_violations": request.access_violations > 0,
            "overdue_tasks": request.tasks_overdue > 2
        }
    }

@app.post("/detect-anomaly")
async def detect_anomaly(features: dict):
    """Anomaly detection for security monitoring"""
    # Convert features to numeric array
    feature_array = {
        'login_time': features.get('login_time', 0),
        'location_change': features.get('location_change', 0),
        'unusual_access': features.get('unusual_access', 0),
        'data_volume': features.get('data_volume', 0)
    }
    
    # Use online learning for anomaly detection
    score = anomaly_detector.score_one(feature_array)
    anomaly_detector.learn_one(feature_array)
    
    is_anomaly = score > 0.7
    
    return {
        "is_anomaly": is_anomaly,
        "anomaly_score": float(score),
        "severity": "high" if score > 0.9 else "medium" if score > 0.7 else "low"
    }

@app.post("/predict-burnout")
async def predict_burnout(request: BurnoutRequest):
    """Burnout detection using workload analysis"""
    # Calculate burnout indicators
    workload_ratio = request.tasks_assigned / max(request.tasks_completed, 1)
    completion_rate = request.tasks_completed / max(request.tasks_assigned, 1)
    
    burnout_score = (
        (workload_ratio - 1) * 0.3 +
        request.missed_deadlines * 0.25 +
        (request.avg_working_hours - 8) * 0.2 +
        (request.time_since_break / 7) * 0.25
    )
    
    burnout_level = 'low'
    if burnout_score > 3:
        burnout_level = 'high'
    elif burnout_score > 1.5:
        burnout_level = 'medium'
    
    recommendations = []
    if request.avg_working_hours > 9:
        recommendations.append("Reduce daily working hours")
    if request.time_since_break > 14:
        recommendations.append("Take a break soon")
    if workload_ratio > 1.5:
        recommendations.append("Redistribute workload")
    
    return {
        "user_id": request.user_id,
        "burnout_score": float(burnout_score),
        "burnout_level": burnout_level,
        "recommendations": recommendations
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Frontend Integration

### API Client Setup

```javascript
// frontend/src/api/client.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_URL = process.env.REACT_APP_ML_URL;

const apiClient = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

const mlClient = axios.create({
  baseURL: ML_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add auth token to requests
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

export { apiClient, mlClient };
```

### Task Dashboard Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { apiClient } from '../api/client';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await apiClient.get('/tasks');
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'inProgress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await apiClient.patch(`/tasks/${taskId}/status`, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-${task.priority}`}>{task.priority}</span>
      <select 
        value={task.status} 
        onChange={(e) => updateTaskStatus(task._id, e.target.value)}
      >
        <option value="todo">To Do</option>
        <option value="inProgress">In Progress</option>
        <option value="done">Done</option>
      </select>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      <div className="column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default TaskBoard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { apiClient, mlClient } from '../api/client';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeTasks: 0,
    openTickets: 0,
    riskAlerts: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const [users, tasks, tickets] = await Promise.all([
        apiClient.get('/admin/users'),
        apiClient.get('/admin/tasks'),
        apiClient.get('/admin/tickets')
      ]);

      // Get risk analysis for each user
      const riskPromises = users.data.map(user =>
        mlClient.post('/analyze-risk', {
          user_id: user._id,
          failed_logins: user.failedLogins || 0,
          tasks_completed: user.tasksCompleted || 0,
          tasks_overdue: user.tasksOverdue || 0,
          avg_completion_time: user.avgCompletionTime || 24,
          access_violations: user.accessViolations || 0
        })
      );

      const riskResults = await Promise.all(riskPromises);
      const highRiskUsers = riskResults
        .filter(r => r.data.risk_level === 'high')
        .map(r => r.data);

      setAnalytics({
        totalUsers: users.data.length,
        activeTasks: tasks.data.filter(t => t.status !== 'done').length,
        openTickets: tickets.data.filter(t => t.status === 'open').length,
        riskAlerts: highRiskUsers
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h2>Admin Dashboard</h2>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-value">{analytics.totalUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p className="stat-value">{analytics.activeTasks}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p className="stat-value">{analytics.openTickets}</p>
        </div>
        <div className="stat-card alert">
          <h3>Risk Alerts</h3>
          <p className="stat-value">{analytics.riskAlerts.length}</p>
        </div>
      </div>

      {analytics.riskAlerts.length > 0 && (
        <div className="risk-alerts">
          <h3>High Risk Users</h3>
          {analytics.riskAlerts.map((alert, idx) => (
            <div key={idx} className="alert-item">
              <span>User ID: {alert.user_id}</span>
              <span>Risk Score: {alert.risk_score.toFixed(2)}</span>
              <span className="badge-high">{alert.risk_level}</span>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

export default AdminDashboard;
```

## Common Patterns

### Ticket Creation with AI Classification

```javascript
// Create and auto-classify support ticket
const createTicket = async (title, description) => {
  try {
    const response = await apiClient.post('/tickets', {
      title,
      description,
      priority: 'medium' // AI will override if needed
    });
    
    console.log('Ticket created:', response.data);
    console.log('AI Category:', response.data.category);
    console.log('AI Priority:', response.data.priority);
    
    return response.data;
  } catch (error) {
    console.error('Error creating ticket:', error);
    throw error;
  }
};
```

### Real-time Burnout Monitoring

```javascript
// Check user burnout status
const checkBurnout = async (userId, userStats) => {
  try {
    const response = await mlClient.post('/predict-burnout', {
      user_id: userId,
      tasks_assigned: userStats.tasksAssigned,
      tasks_completed: userStats.tasksCompleted,
      avg_working_hours: userStats.avgWorkingHours,
      missed_deadlines: userStats.missedDeadlines,
      time_since_break: userStats.daysSinceBreak
    });
    
    if (response.data.burnout_level === 'high') {
      // Show warning to admin
      alert(`User ${userId} showing high burnout risk!`);
      console.log('Recommendations:', response.data.recommendations);
    }
    
    return response.data;
  } catch (error) {
    console.error('Burnout check failed:', error);
  }
};
```

### Anomaly Detection for Security

```javascript
// Monitor user activity for anomalies
const monitorUserActivity = async (activityData) => {
  try {
    const response = await mlClient.post('/detect-anomaly', {
      login_time: activityData.loginHour,
      location_change: activityData.locationChanged ? 1 : 0,
      unusual_access: activityData.accessedSensitiveData ? 1 : 0,
      data_volume: activityData.dataDownloadedMB
    });
    
    if (response.data.is_anomaly) {
      // Log security alert
      console.warn('Anomalous activity detected!');
      console.log('Severity:', response.data.severity);
      console.log('Score:', response.data.anomaly_score);
      
      // Notify admin
      await apiClient.post('/admin/security-alerts', {
        type: 'anomaly',
        severity: response.data.severity,
        details: activityData
      });
    }
    
    return response.data;
  } catch (error) {
    console.error('Anomaly detection error:', error);
  }
};
```

## Configuration

### MongoDB Schema Examples

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  failedLogins: { type: Number, default: 0 },
  tasksCompleted: { type: Number, default: 0 },
  tasksOverdue: { type: Number, default: 0 },
  avgCompletionTime: { type: Number, default: 24 },
  accessViolations: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', userSchema);
```

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'inProgress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  deadline: Date,
  completedAt: Date,
  timeTracking: [{
    date: Date,
    minutes: Number,
    userId: mongoose.Schema.Types.ObjectId
  }],
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

## Troubleshooting

### ML Service Not Connecting

**Issue:** Frontend/Backend cannot reach ML service

**Solution:**
```bash
# Check ML service is running
curl http://localhost:8000/health

# Check CORS settings in ml-service/main.py
# Ensure allow_origins includes your frontend URL

# Verify environment variable
echo $ML_SERVICE_URL  # Should be http://localhost:8000
```

### JWT Authentication Errors

**Issue:** "Invalid token" or "No authentication token"

**Solution:**
```javascript
// Check token storage
console.log('Token:', localStorage.getItem('token'));

// Verify token format in requests
// Should be: Authorization: Bearer <token>

// Check JWT_SECRET matches in .env
// Backend and frontend must use same secret for validation
```

### MongoDB Connection Issues

**Issue:** "MongooseServerSelectionError"

**Solution:**
```bash
# Check MongoDB is running
mongosh  # or mongo

# Verify connection string
# Should be: mongodb://localhost:27017/enterprise_user_mgmt

# Check MongoDB logs
tail -f /var/log/mongodb/mongod.log

# Test connection
mongosh "mongodb://localhost:27017/enterprise
