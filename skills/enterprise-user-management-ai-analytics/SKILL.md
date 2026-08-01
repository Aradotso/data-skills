---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, task tracking, and organizational insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create task tracking with burnout detection"
  - "build admin dashboard with AI insights"
  - "add ticket classification system"
  - "integrate ML service for user analytics"
  - "develop kanban board with time tracking"
  - "implement role-based access control with AI"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack application combining user/task management with AI-powered analytics. It provides role-based access control, Kanban task tracking, support ticketing, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and project delay forecasting.

**Stack**: React frontend, Node.js/Express backend, FastAPI ML service, MongoDB database, JWT authentication.

**AI Features**: Ticket classification, risk scoring, anomaly detection, burnout prediction, project insights.

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+ and pip
- MongoDB instance (local or cloud)

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
JWT_SECRET=your_jwt_secret_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
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
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
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

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Core Architecture

### Backend API Structure

The Node.js backend follows this structure:

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const jwt = require('jsonwebtoken');
require('dotenv').config();

const app = express();

app.use(cors());
app.use(express.json());

// MongoDB connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => console.log('MongoDB connected'))
  .catch(err => console.error('MongoDB connection error:', err));

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/tasks', require('./routes/tasks'));
app.use('/api/tickets', require('./routes/tickets'));
app.use('/api/analytics', require('./routes/analytics'));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const auth = (req, res, next) => {
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

const adminAuth = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied: Admin only' });
  }
  next();
};

module.exports = { auth, adminAuth };
```

## User Management API

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  riskScore: { type: Number, default: 0 },
  anomalyDetected: { type: Boolean, default: false }
});

module.exports = mongoose.model('User', userSchema);
```

### User Routes

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { auth, adminAuth } = require('../middleware/auth');
const bcrypt = require('bcryptjs');

// Get all users (Admin only)
router.get('/', auth, adminAuth, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create user (Admin only)
router.post('/', auth, adminAuth, async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const hashedPassword = await bcrypt.hash(password, 10);
    
    const user = new User({
      name,
      email,
      password: hashedPassword,
      role,
      department
    });
    
    await user.save();
    res.status(201).json({ message: 'User created successfully', userId: user._id });
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Update user
router.put('/:id', auth, adminAuth, async (req, res) => {
  try {
    const { name, email, role, department, status } = req.body;
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { name, email, role, department, status },
      { new: true }
    ).select('-password');
    
    res.json(user);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Delete user
router.delete('/:id', auth, adminAuth, async (req, res) => {
  try {
    await User.findByIdAndDelete(req.params.id);
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## Task Management API

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Task Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { auth } = require('../middleware/auth');

// Get tasks for logged-in user
router.get('/my-tasks', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      createdBy: req.user.id,
      priority,
      dueDate
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Update time spent
router.patch('/:id/time', auth, async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeSpent } },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

module.exports = router;
```

## Support Ticket System

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { type: String, enum: ['technical', 'billing', 'general', 'urgent'], default: 'general' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  status: { type: String, enum: ['open', 'in-progress', 'resolved', 'closed'], default: 'open' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: String,
  aiConfidence: Number,
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### Ticket Routes with AI Integration

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const { auth } = require('../middleware/auth');
const axios = require('axios');

// Create ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Call ML service for classification
    let aiClassification = 'general';
    let aiConfidence = 0;
    
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        title,
        description
      });
      aiClassification = mlResponse.data.category;
      aiConfidence = mlResponse.data.confidence;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      createdBy: req.user.id,
      category: aiClassification,
      aiClassification,
      aiConfidence
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Get user's tickets
router.get('/my-tickets', auth, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .populate('createdBy', 'name email')
      .populate('assignedTo', 'name email')
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service (FastAPI)

### Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from typing import List, Optional
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

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    userId: str
    tasksCompleted: int
    tasksOverdue: int
    avgCompletionTime: float
    ticketsRaised: int
    loginFrequency: int

class BurnoutDetectionRequest(BaseModel):
    userId: str
    weeklyHours: float
    tasksInProgress: int
    overdueCount: int
    averageTaskDuration: float

@app.get("/")
def read_root():
    return {"service": "Enterprise User Management ML Service", "status": "active"}

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """AI-based ticket classification"""
    # Simple keyword-based classification (replace with trained model)
    text = (request.title + " " + request.description).lower()
    
    if any(word in text for word in ['bug', 'error', 'crash', 'not working']):
        category = 'technical'
        confidence = 0.85
    elif any(word in text for word in ['payment', 'invoice', 'billing', 'charge']):
        category = 'billing'
        confidence = 0.80
    elif any(word in text for word in ['urgent', 'critical', 'asap', 'emergency']):
        category = 'urgent'
        confidence = 0.90
    else:
        category = 'general'
        confidence = 0.70
    
    return {
        "category": category,
        "confidence": confidence,
        "suggestedPriority": "high" if category == "urgent" else "medium"
    }

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior"""
    # Risk scoring algorithm
    risk_score = 0
    
    # Overdue tasks increase risk
    if request.tasksOverdue > 5:
        risk_score += 30
    elif request.tasksOverdue > 2:
        risk_score += 15
    
    # Low completion rate increases risk
    if request.tasksCompleted < 10:
        risk_score += 20
    
    # High ticket volume may indicate issues
    if request.ticketsRaised > 10:
        risk_score += 25
    
    # Low login frequency suggests disengagement
    if request.loginFrequency < 5:
        risk_score += 15
    
    # Slow completion time
    if request.avgCompletionTime > 48:  # hours
        risk_score += 10
    
    risk_level = "high" if risk_score > 60 else "medium" if risk_score > 30 else "low"
    
    return {
        "userId": request.userId,
        "riskScore": min(risk_score, 100),
        "riskLevel": risk_level,
        "factors": {
            "overdueTasksImpact": request.tasksOverdue > 2,
            "lowProductivityImpact": request.tasksCompleted < 10,
            "highTicketVolumeImpact": request.ticketsRaised > 10
        }
    }

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    """Detect potential employee burnout"""
    burnout_score = 0
    
    # Excessive working hours
    if request.weeklyHours > 50:
        burnout_score += 40
    elif request.weeklyHours > 40:
        burnout_score += 20
    
    # Too many concurrent tasks
    if request.tasksInProgress > 8:
        burnout_score += 25
    
    # High overdue rate
    if request.overdueCount > 5:
        burnout_score += 20
    
    # Long task durations suggest struggle
    if request.averageTaskDuration > 72:  # hours
        burnout_score += 15
    
    burnout_risk = "high" if burnout_score > 60 else "moderate" if burnout_score > 30 else "low"
    
    return {
        "userId": request.userId,
        "burnoutScore": min(burnout_score, 100),
        "burnoutRisk": burnout_risk,
        "recommendations": [
            "Redistribute workload" if request.tasksInProgress > 8 else None,
            "Review task complexity" if request.averageTaskDuration > 72 else None,
            "Schedule check-in meeting" if burnout_score > 60 else None
        ]
    }

@app.post("/anomaly-detection")
async def detect_anomaly(userId: str, loginTime: str, ipAddress: str, location: Optional[str] = None):
    """Detect anomalous user behavior"""
    # Simple anomaly detection (replace with trained model)
    anomaly_detected = False
    confidence = 0.0
    
    # Check for unusual login times (e.g., 2 AM - 5 AM)
    from datetime import datetime
    try:
        hour = datetime.fromisoformat(loginTime).hour
        if 2 <= hour <= 5:
            anomaly_detected = True
            confidence = 0.75
    except:
        pass
    
    return {
        "userId": userId,
        "anomalyDetected": anomaly_detected,
        "confidence": confidence,
        "reason": "Unusual login time" if anomaly_detected else "Normal activity"
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Requirements File

```txt
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
numpy==1.24.3
scikit-learn==1.3.2
joblib==1.3.2
python-dotenv==1.0.0
river==0.18.0
pandas==2.0.3
```

## Frontend Integration

### API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

const getAuthHeader = () => ({
  headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
});

export const authAPI = {
  login: (email, password) => axios.post(`${API_URL}/api/auth/login`, { email, password }),
  register: (userData) => axios.post(`${API_URL}/api/auth/register`, userData),
  getProfile: () => axios.get(`${API_URL}/api/auth/profile`, getAuthHeader())
};

export const userAPI = {
  getAll: () => axios.get(`${API_URL}/api/users`, getAuthHeader()),
  create: (userData) => axios.post(`${API_URL}/api/users`, userData, getAuthHeader()),
  update: (id, userData) => axios.put(`${API_URL}/api/users/${id}`, userData, getAuthHeader()),
  delete: (id) => axios.delete(`${API_URL}/api/users/${id}`, getAuthHeader())
};

export const taskAPI = {
  getMyTasks: () => axios.get(`${API_URL}/api/tasks/my-tasks`, getAuthHeader()),
  create: (taskData) => axios.post(`${API_URL}/api/tasks`, taskData, getAuthHeader()),
  updateStatus: (id, status) => axios.patch(`${API_URL}/api/tasks/${id}/status`, { status }, getAuthHeader()),
  updateTime: (id, timeSpent) => axios.patch(`${API_URL}/api/tasks/${id}/time`, { timeSpent }, getAuthHeader())
};

export const ticketAPI = {
  getMyTickets: () => axios.get(`${API_URL}/api/tickets/my-tickets`, getAuthHeader()),
  create: (ticketData) => axios.post(`${API_URL}/api/tickets`, ticketData, getAuthHeader())
};

export const mlAPI = {
  classifyTicket: (title, description) => 
    axios.post(`${ML_API_URL}/classify-ticket`, { title, description }),
  predictRisk: (userData) => 
    axios.post(`${ML_API_URL}/predict-risk`, userData),
  detectBurnout: (userData) => 
    axios.post(`${ML_API_URL}/detect-burnout`, userData)
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskAPI.getMyTasks();
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status].push(task);
        return acc;
      }, { todo: [], 'in-progress': [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskAPI.updateStatus(taskId, newStatus);
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const renderColumn = (status, title) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      {tasks[status].map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <div className="task-actions">
            {status !== 'done' && (
              <button onClick={() => handleStatusChange(task._id, 
                status === 'todo' ? 'in-progress' : 'done')}>
                Move to {status === 'todo' ? 'In Progress' : 'Done'}
              </button>
            )}
          </div>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do')}
      {renderColumn('in-progress', 'In Progress')}
      {renderColumn('done', 'Done')}
    </div>
  );
};

export default KanbanBoard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { userAPI, mlAPI } from '../services/api';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState({});

  useEffect(() => {
    fetchUsers();
  }, []);

  const fetchUsers = async () => {
    try {
      const response = await userAPI.getAll();
      setUsers(response.data);
      analyzeUsers(response.data);
    } catch (error) {
      console.error('Failed to fetch users:', error);
    }
  };

  const analyzeUsers = async (usersList) => {
    const riskPromises = usersList.map(async (user) => {
      try {
        const riskData = {
          userId: user._id,
          tasksCompleted: user.tasksCompleted || 0,
          tasksOverdue: user.tasksOverdue || 0,
          avgCompletionTime: user.avgCompletionTime || 24,
          ticketsRaised: user.ticketsRaised || 0,
          loginFrequency: user.loginFrequency || 5
        };
        const response = await mlAPI.predictRisk(riskData);
        return { userId: user._id, ...response.data };
      } catch (error) {
        return null;
      }
    });

    const risks = await Promise.all(riskPromises);
    const highRiskUsers = risks.filter(r => r && r.riskLevel === 'high');
    
    setAnalytics({
      totalUsers: usersList.length,
      activeUsers: usersList.filter(u => u.status === 'active').length,
      highRiskUsers: highRiskUsers.length,
      risks
    });
  };

  return (
    <div className="admin-dashboard">
      <h2>Admin Analytics Dashboard</h2>
      
      <div className="metrics">
        <div className="metric-card">
          <h3>Total Users</h3>
          <p>{analytics.totalUsers || 0}</p>
        </div>
        <div className="metric-card">
          <h3>Active Users</h3>
          <p>{analytics.activeUsers || 0}</p>
        </div>
        <div className="metric-card alert">
          <h3>High Risk Users</h3>
          <p>{analytics.highRiskUsers || 0}</p>
        </div>
      </div>

      <div className="user-list">
        <h3>User Management</h3>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Risk Level</th>
              <th>Status</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => {
              const userRisk = analytics.risks?.find(r => r?.userId === user._id);
              return (
                <tr key={user._id}>
                  <td>{user.name}</td>
                  <td>{user.email}</td>
                  <td>{user.role}</td>
                  <td className={`risk-${userRisk?.riskLevel || 'low'}`}>
                    {userRisk?.riskLevel || 'N/A'}
                  </td>
                  <td>{user.status}</td>
                </tr>
              );
            })}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Configuration

### Environment Variables

**Backend (.env)**:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_secure_jwt_secret_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**:
```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
PYTHONUNBUFFERED=1
```

**Frontend (.env)**:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Common Patterns

### Creating a New User (Admin)

```javascript
import { userAPI } from './services/api';

const createUser = async () => {
  try {
    const userData = {
      name: 'John Doe',
      email: 'john@example.com',
      password: 'securePassword123',
      role: 'user',
      department: 'Engineering'
    };
    
    const response = await userAPI.create(userData);
    console.log('User created:', response.data);
  } catch (error) {
    console.error('Error creating user:', error.response?.data?.message);
  }
};
```

### Submitting a Ticket with AI Classification

```javascript
import { ticketAPI } from './services/api';

const submitTicket = async () => {
  try {
    const ticketData = {
      title: 'Login not working',
      description: 'I am getting an error when trying to log in to the system'
    };
    
    const response = await ticketAPI.create(ticketData);
    console.log('Ticket created:', response.data);
    console.log('AI classified as:', response.data.category);
  } catch (error) {
    console.error('Error submitting ticket:', error);
  }
};
```

### Detecting User Burnout

```python
# Direct ML service call
import requests

def check_burnout(user_id):
    data = {
        "userId": user_id,
        "weeklyHours": 55.0,
        "tasksInProgress": 10,
        "overdueCount": 6,
        "averageTaskDuration": 80.0
    }
    
    response = requests
