---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management built with React, Node.js, and FastAPI ML service
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with risk detection"
  - "integrate machine learning for burnout prediction"
  - "build task management with AI insights"
  - "configure user management system with AI analytics"
  - "add anomaly detection to user management"
  - "implement ticket classification with AI"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack application for managing enterprise users, tasks, and support tickets with integrated AI analytics. The system provides risk detection, anomaly identification, burnout analysis, and predictive project insights using machine learning models. Built with React frontend, Node.js/Express backend, and FastAPI ML service.

## What This Project Does

This system enables organizations to:
- Manage users with role-based access control (Admin/User)
- Track tasks using Kanban boards with time tracking
- Handle support tickets with AI-powered classification and routing
- Detect security anomalies and suspicious user behavior
- Predict employee burnout based on workload analysis
- Forecast project delays using predictive analytics
- Provide intelligent insights through an AI assistant

The architecture consists of three main services:
1. **Frontend** (React): User interface for dashboards, task management, and ticket tracking
2. **Backend** (Node.js/Express): REST API for user management, authentication, and business logic
3. **ML Service** (FastAPI): AI/ML endpoints for predictions, classification, and analytics

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance running

### Clone and Setup

```bash
# Clone repository
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
JWT_EXPIRE=24h
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file for ML service
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

ML service runs at `http://localhost:8000`

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

Frontend runs at `http://localhost:3000`

## Key API Endpoints

### Backend REST API

**Authentication**
```javascript
// POST /api/auth/register - Register new user
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepass123",
  "role": "user"
}

// POST /api/auth/login - Login
{
  "email": "john@example.com",
  "password": "securepass123"
}
// Returns: { token: "jwt-token", user: {...} }
```

**User Management**
```javascript
// GET /api/users - Get all users (Admin only)
// GET /api/users/:id - Get user by ID
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user (Admin only)
```

**Task Management**
```javascript
// GET /api/tasks - Get all tasks
// POST /api/tasks - Create task
{
  "title": "Complete documentation",
  "description": "Write API documentation",
  "assignedTo": "userId",
  "status": "todo",
  "priority": "high",
  "dueDate": "2026-05-01"
}

// PUT /api/tasks/:id - Update task status
// DELETE /api/tasks/:id - Delete task
```

**Support Tickets**
```javascript
// GET /api/tickets - Get all tickets
// POST /api/tickets - Create ticket
{
  "subject": "Login issue",
  "description": "Cannot login to system",
  "priority": "high",
  "category": "technical"
}

// PUT /api/tickets/:id - Update ticket
// GET /api/tickets/user/:userId - Get user's tickets
```

### ML Service API

**Risk Prediction**
```python
# POST /api/ml/predict-risk
{
  "userId": "user123",
  "loginAttempts": 5,
  "failedLogins": 3,
  "taskCompletionRate": 0.45,
  "avgResponseTime": 48,
  "ticketsRaised": 12
}
# Returns: { "riskScore": 0.78, "riskLevel": "high", "factors": [...] }
```

**Anomaly Detection**
```python
# POST /api/ml/detect-anomaly
{
  "userId": "user123",
  "loginTime": "2026-04-15T03:30:00Z",
  "ipAddress": "192.168.1.100",
  "location": "unusual_location",
  "deviceId": "new_device"
}
# Returns: { "isAnomaly": true, "anomalyScore": 0.92, "reasons": [...] }
```

**Burnout Prediction**
```python
# POST /api/ml/predict-burnout
{
  "userId": "user123",
  "hoursWorked": 65,
  "tasksAssigned": 25,
  "tasksCompleted": 12,
  "overtimeHours": 15,
  "ticketsRaised": 8
}
# Returns: { "burnoutRisk": 0.85, "recommendation": "Reduce workload", "factors": [...] }
```

**Ticket Classification**
```python
# POST /api/ml/classify-ticket
{
  "subject": "Cannot access database",
  "description": "Getting connection timeout errors",
  "priority": "high"
}
# Returns: { "category": "technical", "suggestedAssignee": "techTeam", "confidence": 0.94 }
```

## Code Examples

### Backend: Creating Protected Routes with JWT

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = decoded.userId;
    req.userRole = decoded.role;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// middleware/admin.js
const adminMiddleware = (req, res, next) => {
  if (req.userRole !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Backend: User Controller with AI Integration

```javascript
// controllers/userController.js
const axios = require('axios');

const getUserRiskAnalysis = async (req, res) => {
  try {
    const { userId } = req.params;
    
    // Fetch user activity data
    const user = await User.findById(userId);
    const tasks = await Task.find({ assignedTo: userId });
    const tickets = await Ticket.find({ createdBy: userId });
    
    // Prepare data for ML service
    const riskData = {
      userId: user._id,
      loginAttempts: user.loginAttempts || 0,
      failedLogins: user.failedLogins || 0,
      taskCompletionRate: calculateCompletionRate(tasks),
      avgResponseTime: calculateAvgResponseTime(tickets),
      ticketsRaised: tickets.length
    };
    
    // Call ML service
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/api/ml/predict-risk`,
      riskData
    );
    
    res.json({
      user: user.name,
      riskAnalysis: mlResponse.data
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

const calculateCompletionRate = (tasks) => {
  if (tasks.length === 0) return 0;
  const completed = tasks.filter(t => t.status === 'done').length;
  return completed / tasks.length;
};

module.exports = { getUserRiskAnalysis };
```

### Backend: Task Routes

```javascript
// routes/taskRoutes.js
const express = require('express');
const router = express.Router();
const { authMiddleware, adminMiddleware } = require('../middleware/auth');
const Task = require('../models/Task');

// Get all tasks for logged-in user
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = req.userRole === 'admin' 
      ? await Task.find().populate('assignedTo', 'name email')
      : await Task.find({ assignedTo: req.userId });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create new task
router.post('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.userId
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update task status
router.put('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status, timeSpent } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { 
        status,
        $inc: { totalTimeSpent: timeSpent || 0 },
        lastUpdated: Date.now()
      },
      { new: true }
    );
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### ML Service: Risk Prediction Model

```python
# ml-service/services/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
import os

class RiskPredictor:
    def __init__(self):
        self.model = None
        self.load_model()
    
    def load_model(self):
        model_path = os.getenv('MODEL_PATH', './models')
        try:
            self.model = joblib.load(f"{model_path}/risk_model.pkl")
        except FileNotFoundError:
            # Initialize and train a basic model if not found
            self.model = RandomForestClassifier(n_estimators=100, random_state=42)
            self._train_default_model()
    
    def _train_default_model(self):
        # Training with synthetic data - replace with real data in production
        X = np.random.rand(1000, 5)
        y = (X[:, 1] > 0.6) | (X[:, 2] < 0.4)  # Simple rule-based labels
        self.model.fit(X, y)
    
    def predict_risk(self, user_data):
        features = self._extract_features(user_data)
        risk_proba = self.model.predict_proba([features])[0][1]
        
        risk_level = self._determine_risk_level(risk_proba)
        factors = self._identify_risk_factors(user_data, features)
        
        return {
            "riskScore": float(risk_proba),
            "riskLevel": risk_level,
            "factors": factors
        }
    
    def _extract_features(self, data):
        return [
            data.get('loginAttempts', 0) / 10.0,
            data.get('failedLogins', 0) / 5.0,
            1 - data.get('taskCompletionRate', 1.0),
            data.get('avgResponseTime', 24) / 72.0,
            data.get('ticketsRaised', 0) / 20.0
        ]
    
    def _determine_risk_level(self, score):
        if score > 0.7:
            return "high"
        elif score > 0.4:
            return "medium"
        return "low"
    
    def _identify_risk_factors(self, data, features):
        factors = []
        if data.get('failedLogins', 0) > 3:
            factors.append("Multiple failed login attempts")
        if data.get('taskCompletionRate', 1.0) < 0.5:
            factors.append("Low task completion rate")
        if data.get('avgResponseTime', 24) > 48:
            factors.append("Slow response time")
        if data.get('ticketsRaised', 0) > 15:
            factors.append("High number of support tickets")
        return factors

risk_predictor = RiskPredictor()
```

### ML Service: FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
from services.risk_predictor import risk_predictor
from services.anomaly_detector import anomaly_detector
from services.burnout_predictor import burnout_predictor
from services.ticket_classifier import ticket_classifier

app = FastAPI(title="Enterprise User Management AI Service")

class RiskPredictionRequest(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    taskCompletionRate: float
    avgResponseTime: float
    ticketsRaised: int

class AnomalyDetectionRequest(BaseModel):
    userId: str
    loginTime: str
    ipAddress: str
    location: str
    deviceId: str

class BurnoutPredictionRequest(BaseModel):
    userId: str
    hoursWorked: float
    tasksAssigned: int
    tasksCompleted: int
    overtimeHours: float
    ticketsRaised: int

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str
    priority: str

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        result = risk_predictor.predict_risk(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        result = anomaly_detector.detect(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-burnout")
async def predict_burnout(request: BurnoutPredictionRequest):
    try:
        result = burnout_predictor.predict(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        result = ticket_classifier.classify(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

### Frontend: React Dashboard Component

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      const [usersRes, tasksRes, ticketsRes] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/api/users`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/api/tickets`, config)
      ]);

      setUsers(usersRes.data);
      
      const analyticsData = {
        totalUsers: usersRes.data.length,
        activeTasks: tasksRes.data.filter(t => t.status !== 'done').length,
        openTickets: ticketsRes.data.filter(t => t.status === 'open').length,
        completedTasks: tasksRes.data.filter(t => t.status === 'done').length
      };
      
      setAnalytics(analyticsData);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
      setLoading(false);
    }
  };

  const analyzeUserRisk = async (userId) => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/users/${userId}/risk`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      alert(`Risk Analysis:\nRisk Level: ${response.data.riskAnalysis.riskLevel}\nScore: ${response.data.riskAnalysis.riskScore}`);
    } catch (error) {
      console.error('Error analyzing user risk:', error);
    }
  };

  if (loading) return <div>Loading dashboard...</div>;

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="analytics-cards">
        <div className="card">
          <h3>Total Users</h3>
          <p className="stat">{analytics.totalUsers}</p>
        </div>
        <div className="card">
          <h3>Active Tasks</h3>
          <p className="stat">{analytics.activeTasks}</p>
        </div>
        <div className="card">
          <h3>Open Tickets</h3>
          <p className="stat">{analytics.openTickets}</p>
        </div>
        <div className="card">
          <h3>Completed Tasks</h3>
          <p className="stat">{analytics.completedTasks}</p>
        </div>
      </div>

      <div className="users-list">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.name}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>
                  <button onClick={() => analyzeUserRisk(user._id)}>
                    Risk Analysis
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AdminDashboard;
```

### Frontend: Task Management with Time Tracking

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [activeTimer, setActiveTimer] = useState(null);
  const [elapsedTime, setElapsedTime] = useState(0);

  useEffect(() => {
    fetchTasks();
  }, []);

  useEffect(() => {
    let interval;
    if (activeTimer) {
      interval = setInterval(() => {
        setElapsedTime(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [activeTimer]);

  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks`,
        { headers: { Authorization: `Bearer ${token}` } }
      );

      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };

      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.put(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus, timeSpent: elapsedTime },
        { headers: { Authorization: `Bearer ${token}` } }
      );

      if (activeTimer === taskId) {
        stopTimer();
      }

      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const startTimer = (taskId) => {
    setActiveTimer(taskId);
    setElapsedTime(0);
  };

  const stopTimer = () => {
    setActiveTimer(null);
    setElapsedTime(0);
  };

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const renderColumn = (columnTasks, status, title) => (
    <div className="task-column">
      <h3>{title}</h3>
      {columnTasks.map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority-${task.priority}`}>
            {task.priority}
          </span>
          
          {status === 'in-progress' && (
            <div className="timer">
              {activeTimer === task._id ? (
                <>
                  <p>{formatTime(elapsedTime)}</p>
                  <button onClick={stopTimer}>Stop</button>
                </>
              ) : (
                <button onClick={() => startTimer(task._id)}>
                  Start Timer
                </button>
              )}
            </div>
          )}

          <div className="task-actions">
            {status !== 'done' && (
              <button onClick={() => updateTaskStatus(
                task._id,
                status === 'todo' ? 'in-progress' : 'done'
              )}>
                {status === 'todo' ? 'Start' : 'Complete'}
              </button>
            )}
          </div>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn(tasks.todo, 'todo', 'To Do')}
      {renderColumn(tasks.inProgress, 'in-progress', 'In Progress')}
      {renderColumn(tasks.done, 'done', 'Done')}
    </div>
  );
};

export default TaskBoard;
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your-secret-key-min-32-chars
JWT_EXPIRE=24h
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
NODE_ENV=production
```

**ML Service (.env)**
```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
MAX_WORKERS=4
CACHE_ENABLED=true
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENV=development
```

### MongoDB Schema Examples

```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  loginAttempts: { type: Number, default: 0 },
  failedLogins: { type: Number, default: 0 },
  lastLogin: { type: Date },
  isActive: { type: Boolean, default: true },
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', userSchema);

// models/Task.js
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
  totalTimeSpent: { type: Number, default: 0 }, // in seconds
  dueDate: Date,
  createdAt: { type: Date, default: Date.now },
  lastUpdated: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

## Common Patterns

### Authentication Flow

```javascript
// Login and store token
const login = async (email, password) => {
  const response = await axios.post(
    `${process.env.REACT_APP_API_URL}/api/auth/login`,
    { email, password }
  );
  
  localStorage.setItem('token', response.data.token);
  localStorage.setItem('userRole', response.data.user.role);
  return response.data.user;
};

// Create authenticated axios instance
const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

api.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Use in components
const fetchProtectedData = async () => {
  const response = await api.get('/api/users');
  return response.data;
};
```

### Integrating AI Predictions

```javascript
// Fetch and display burnout risk
const checkBurnoutRisk = async (userId) => {
  const userMetrics = await api.get(`/api/users/${userId}/metrics`);
  
  const burnoutData = {
    userId,
    hoursWorked: userMetrics.data.hoursWorked,
    tasksAssigned: userMetrics.data.tasksAssigned,
    tasksCompleted: userMetrics.data.tasksCompleted,
    overtimeHours: userMetrics.data.overtimeHours,
    ticketsRaised: userMetrics.data.ticketsRaised
  };
  
  const mlResponse = await axios.post(
    `${process.env.REACT_APP_ML_API_URL}/api/ml/predict-burnout`,
    burnoutData
  );
  
  return mlResponse.data;
};
```

### Real-time Notifications

```javascript
// Backend: Send notification on ticket creation
const createTicket = async (req, res) => {
  const ticket = new Ticket(req.body);
  await ticket.save();
  
  // Classify ticket using ML
  const classification = await axios.post(
    `${process.env.ML_SERVICE_URL}/api/ml/classify-ticket`,
    {
      subject: ticket.subject,
      description: ticket.description,
      priority: ticket.priority
    }
  );
  
  // Update ticket with classification
  ticket.category = classification.data.category;
  ticket.assignedTo = classification.data.suggestedAssignee;
  await ticket.save();
  
  res.status(201).json(ticket);
};
```

## Troubleshooting

### Common Issues

**MongoDB Connection Errors**
```bash
# Check MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Verify connection string in .env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
```

**JWT Authentication Fails**
```javascript
// Ensure JWT_SECRET is set and consistent
// Backend .env
JWT_SECRET=min-32-character-secret-key-here

// Check token expiration
JWT_EXPIRE=24h  // Adjust as needed

// Frontend: Handle expired tokens
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
```

**ML Service Not Responding**
```bash
# Check if ML service is running
curl http://localhost:8000/health

# Check Python dependencies
cd ml-service
pip install -r requirements.txt

# Run with detailed logs
uvicorn main:
