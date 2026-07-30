```yaml
---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered task analytics, risk detection, and ticket classification
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered ticket classification"
  - "create user management dashboard with analytics"
  - "integrate burnout detection for users"
  - "build task tracking with anomaly detection"
  - "configure AI risk prediction for enterprise"
  - "deploy user management with ML service"
  - "implement Kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application combining user/task management with AI-driven analytics. It provides:

- **User Management**: JWT authentication, role-based access control (RBAC), user CRUD operations
- **Task Tracking**: Kanban boards (To Do → In Progress → Done), time tracking, assignment management
- **Support Tickets**: Create, classify, route, and track support requests
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, predictive project insights
- **Dashboards**: Admin analytics, user performance metrics, audit logs

**Architecture**: React frontend + Node.js/Express backend + FastAPI ML service + MongoDB database

## Installation

### Prerequisites

```bash
# Node.js 14+ and Python 3.8+
node --version
python --version
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
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=info
EOF

uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
EOF

npm start
```

## Backend API (Node.js/Express)

### User Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

const router = express.Router();

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
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
      { expiresIn: '7d' }
    );
    
    res.json({ token, user: { id: user._id, username, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
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
    return res.status(401).json({ error: 'No authentication token' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

const router = express.Router();

// Get user tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('assignedBy', 'username')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.userId,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body; // duration in seconds
    const task = await Task.findById(req.params.id);
    
    task.timeSpent = (task.timeSpent || 0) + duration;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Ticket API

```javascript
// backend/routes/tickets.js
const express = require('express');
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authMiddleware } = require('../middleware/auth');

const router = express.Router();

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      { title, description }
    );
    
    const ticket = new Ticket({
      title,
      description,
      priority: priority || mlResponse.data.priority,
      category: mlResponse.data.category,
      status: 'open',
      createdBy: req.user.userId,
      assignedTo: mlResponse.data.suggestedAssignee
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.userId })
      .populate('assignedTo', 'username')
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service API (FastAPI + scikit-learn)

### FastAPI Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Dict
import os

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    userId: str
    taskCount: int
    completionRate: float
    avgResponseTime: float
    failedLoginAttempts: int

class BurnoutDetectionRequest(BaseModel):
    userId: str
    tasksCompleted: int
    hoursWorked: float
    overtimeHours: float
    missedDeadlines: int
    weeksSinceBreak: int

@app.get("/")
def root():
    return {"service": "Enterprise User Management ML", "version": "1.0"}
```

### Ticket Classification

```python
# ml-service/main.py (continued)
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import pickle

# Initialize ticket classifier
ticket_vectorizer = TfidfVectorizer(max_features=1000)
ticket_model = MultinomialNB()

CATEGORIES = ["technical", "billing", "access", "general", "urgent"]
PRIORITIES = ["low", "medium", "high", "critical"]

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket using NLP"""
    try:
        text = f"{request.title} {request.description}"
        
        # Simple rule-based classification (replace with trained model)
        keywords = {
            "technical": ["bug", "error", "crash", "not working", "broken"],
            "billing": ["payment", "invoice", "charge", "subscription"],
            "access": ["login", "password", "permission", "locked"],
            "urgent": ["urgent", "critical", "emergency", "down"]
        }
        
        text_lower = text.lower()
        category = "general"
        priority = "medium"
        
        for cat, terms in keywords.items():
            if any(term in text_lower for term in terms):
                category = cat
                if cat == "urgent":
                    priority = "critical"
                break
        
        # Suggest assignee based on category
        assignee_mapping = {
            "technical": "tech-team-id",
            "billing": "finance-team-id",
            "access": "admin-team-id"
        }
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85,
            "suggestedAssignee": assignee_mapping.get(category)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Risk Prediction

```python
# ml-service/main.py (continued)
from sklearn.ensemble import RandomForestClassifier

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior patterns"""
    try:
        # Feature engineering
        features = np.array([[
            request.taskCount,
            request.completionRate,
            request.avgResponseTime,
            request.failedLoginAttempts
        ]])
        
        # Simple risk scoring (replace with trained model)
        risk_score = 0.0
        
        if request.completionRate < 0.5:
            risk_score += 0.3
        if request.failedLoginAttempts > 3:
            risk_score += 0.4
        if request.avgResponseTime > 24:  # hours
            risk_score += 0.2
        
        risk_level = "low"
        if risk_score > 0.7:
            risk_level = "high"
        elif risk_score > 0.4:
            risk_level = "medium"
        
        return {
            "userId": request.userId,
            "riskScore": min(risk_score, 1.0),
            "riskLevel": risk_level,
            "factors": {
                "lowCompletionRate": request.completionRate < 0.5,
                "suspiciousLogin": request.failedLoginAttempts > 3,
                "slowResponse": request.avgResponseTime > 24
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Burnout Detection

```python
# ml-service/main.py (continued)
@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    """Detect employee burnout using workload analysis"""
    try:
        # Calculate burnout indicators
        burnout_score = 0.0
        
        # High workload
        if request.hoursWorked > 50:
            burnout_score += 0.25
        
        # Excessive overtime
        if request.overtimeHours > 10:
            burnout_score += 0.25
        
        # Missed deadlines
        if request.missedDeadlines > 2:
            burnout_score += 0.2
        
        # No breaks
        if request.weeksSinceBreak > 8:
            burnout_score += 0.3
        
        burnout_level = "low"
        if burnout_score > 0.7:
            burnout_level = "high"
        elif burnout_score > 0.4:
            burnout_level = "medium"
        
        recommendations = []
        if request.overtimeHours > 10:
            recommendations.append("Reduce overtime hours")
        if request.weeksSinceBreak > 8:
            recommendations.append("Schedule time off")
        if request.missedDeadlines > 2:
            recommendations.append("Reassess workload distribution")
        
        return {
            "userId": request.userId,
            "burnoutScore": min(burnout_score, 1.0),
            "burnoutLevel": burnout_level,
            "recommendations": recommendations,
            "metrics": {
                "hoursWorked": request.hoursWorked,
                "overtimeHours": request.overtimeHours,
                "weeksSinceBreak": request.weeksSinceBreak
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Anomaly Detection

```python
# ml-service/main.py (continued)
from river import anomaly
from river import stream

# Online learning anomaly detector
anomaly_detector = anomaly.HalfSpaceTrees()

@app.post("/detect-anomaly")
async def detect_anomaly(data: Dict):
    """Detect anomalies in user behavior"""
    try:
        # Extract features
        features = {
            "loginTime": data.get("loginTime", 9),
            "activeHours": data.get("activeHours", 8),
            "locationChanges": data.get("locationChanges", 0),
            "apiCallsPerHour": data.get("apiCallsPerHour", 10)
        }
        
        # Get anomaly score
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.7
        
        return {
            "isAnomaly": is_anomaly,
            "score": float(score),
            "features": features,
            "alert": "Unusual behavior detected" if is_anomaly else None
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend (React)

### API Client Setup

```javascript
// frontend/src/api/client.js
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to requests
apiClient.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default apiClient;
```

### User Dashboard Component

```javascript
// frontend/src/components/UserDashboard.js
import React, { useEffect, useState } from 'react';
import apiClient from '../api/client';
import KanbanBoard from './KanbanBoard';
import TimeTracker from './TimeTracker';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [burnoutData, setBurnoutData] = useState(null);
  
  useEffect(() => {
    fetchTasks();
    checkBurnout();
  }, []);
  
  const fetchTasks = async () => {
    try {
      const response = await apiClient.get('/api/tasks');
      setTasks(response.data);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };
  
  const checkBurnout = async () => {
    try {
      const mlResponse = await axios.post(
        `${process.env.REACT_APP_ML_SERVICE_URL}/detect-burnout`,
        {
          userId: localStorage.getItem('userId'),
          tasksCompleted: 25,
          hoursWorked: 52,
          overtimeHours: 12,
          missedDeadlines: 3,
          weeksSinceBreak: 10
        }
      );
      setBurnoutData(mlResponse.data);
    } catch (error) {
      console.error('Burnout check failed:', error);
    }
  };
  
  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {burnoutData && burnoutData.burnoutLevel === 'high' && (
        <div className="alert alert-warning">
          <strong>Burnout Alert:</strong> {burnoutData.recommendations.join(', ')}
        </div>
      )}
      
      <KanbanBoard tasks={tasks} onUpdate={fetchTasks} />
      <TimeTracker tasks={tasks} />
    </div>
  );
};

export default UserDashboard;
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React from 'react';
import apiClient from '../api/client';
import './KanbanBoard.css';

const KanbanBoard = ({ tasks, onUpdate }) => {
  const columns = {
    todo: tasks.filter(t => t.status === 'todo'),
    inProgress: tasks.filter(t => t.status === 'in-progress'),
    done: tasks.filter(t => t.status === 'done')
  };
  
  const updateStatus = async (taskId, newStatus) => {
    try {
      await apiClient.patch(`/api/tasks/${taskId}/status`, {
        status: newStatus
      });
      onUpdate();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };
  
  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-badge priority-${task.priority}`}>
        {task.priority}
      </span>
      <div className="task-actions">
        {task.status !== 'done' && (
          <button onClick={() => updateStatus(task._id, 
            task.status === 'todo' ? 'in-progress' : 'done'
          )}>
            Move →
          </button>
        )}
      </div>
    </div>
  );
  
  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do ({columns.todo.length})</h3>
        {columns.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      
      <div className="kanban-column">
        <h3>In Progress ({columns.inProgress.length})</h3>
        {columns.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      
      <div className="kanban-column">
        <h3>Done ({columns.done.length})</h3>
        {columns.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.js
import React, { useEffect, useState } from 'react';
import apiClient from '../api/client';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskAnalysis, setRiskAnalysis] = useState({});
  
  useEffect(() => {
    fetchUsers();
  }, []);
  
  const fetchUsers = async () => {
    try {
      const response = await apiClient.get('/api/admin/users');
      setUsers(response.data);
      
      // Analyze risk for each user
      for (const user of response.data) {
        analyzeUserRisk(user);
      }
    } catch (error) {
      console.error('Failed to fetch users:', error);
    }
  };
  
  const analyzeUserRisk = async (user) => {
    try {
      const mlResponse = await axios.post(
        `${process.env.REACT_APP_ML_SERVICE_URL}/predict-risk`,
        {
          userId: user._id,
          taskCount: user.taskCount || 0,
          completionRate: user.completionRate || 1.0,
          avgResponseTime: user.avgResponseTime || 8,
          failedLoginAttempts: user.failedLoginAttempts || 0
        }
      );
      
      setRiskAnalysis(prev => ({
        ...prev,
        [user._id]: mlResponse.data
      }));
    } catch (error) {
      console.error('Risk analysis failed:', error);
    }
  };
  
  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="users-table">
        <table>
          <thead>
            <tr>
              <th>Username</th>
              <th>Email</th>
              <th>Role</th>
              <th>Tasks</th>
              <th>Risk Level</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.username}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.taskCount || 0}</td>
                <td>
                  <span className={`risk-badge risk-${riskAnalysis[user._id]?.riskLevel}`}>
                    {riskAnalysis[user._id]?.riskLevel || 'analyzing...'}
                  </span>
                </td>
                <td>
                  <button>Edit</button>
                  <button>View Details</button>
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

## MongoDB Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  taskCount: { type: Number, default: 0 },
  completionRate: { type: Number, default: 1.0 },
  avgResponseTime: { type: Number, default: 8 },
  failedLoginAttempts: { type: Number, default: 0 },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

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
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  timeSpent: { type: Number, default: 0 }, // in seconds
  dueDate: Date,
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'] 
  },
  category: { 
    type: String, 
    enum: ['technical', 'billing', 'access', 'general', 'urgent'] 
  },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### Protecting Routes with JWT

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const authRoutes = require('./routes/auth');
const taskRoutes = require('./routes/tasks');
const ticketRoutes = require('./routes/tickets');
const { authMiddleware, adminMiddleware } = require('./middleware/auth');

const app = express();

app.use(cors());
app.use(express.json());

// Public routes
app.use('/api/auth', authRoutes);

// Protected routes
app.use('/api/tasks', authMiddleware, taskRoutes);
app.use('/api/tickets', authMiddleware, ticketRoutes);

// Admin-only routes
app.use('/api/admin', authMiddleware, adminMiddleware, require('./routes/admin'));

mongoose.connect(process.env.MONGODB_URI)
  .then(() => console.log('MongoDB connected'))
  .catch(err => console.error('MongoDB error:', err));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### Calling ML Service from Backend

```javascript
// backend/services/mlService.js
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL || 'http://localhost:8000';

const classifyTicket = async (title, description) => {
  const response = await axios.post(`${ML_SERVICE_URL}/classify-ticket`, {
    title,
    description
  });
  return response.data;
};

const predictRisk = async (userData) => {
  const response = await axios.post(`${ML_SERVICE_URL}/predict-risk`, userData);
  return response.data;
};

const detectBurnout = async (userMetrics) => {
  const response = await axios.post(`${ML_SERVICE_URL}/detect-burnout`, userMetrics);
  return response.data;
};

module.exports = { classifyTicket, predictRisk, detectBurnout };
```

### React Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import apiClient from '../api/client';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    const token = localStorage.getItem('token');
    const userData = localStorage.getItem('user');
    
    if (token && userData) {
      setUser(JSON.parse(userData));
    }
    setLoading(false);
  }, []);
  
  const login = async (email, password) => {
    const response = await apiClient.post('/api/auth/login', { email, password });
    localStorage.setItem
