---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and task automation
triggers:
  - "set up enterprise user management with AI analytics"
  - "create a user management dashboard with task tracking"
  - "implement AI-based risk detection for user activity"
  - "build a kanban board with time tracking"
  - "add burnout detection and anomaly analysis"
  - "configure JWT authentication for admin and user roles"
  - "integrate FastAPI ML service with React frontend"
  - "deploy user management system with AI insights"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

The Enterprise User Management System is a full-stack application that combines user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control (admin/user), secure JWT authentication
- **Task Management**: Kanban boards, time tracking, task assignment and monitoring
- **Support System**: Ticket creation, AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Separate admin and user interfaces with real-time insights

**Architecture**: React frontend + Node.js/Express backend + FastAPI ML service + MongoDB database

## Installation

### Prerequisites

```bash
# Required
node --version  # v14+ required
python --version  # Python 3.8+ required
mongod --version  # MongoDB 4.4+ required
```

### Full Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
npm start  # Runs on http://localhost:5000

# ML Service setup (new terminal)
cd ml-service
pip install -r requirements.txt
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup (new terminal)
cd frontend
npm install
npm start  # Runs on http://localhost:3000
```

### Environment Configuration

**Backend (.env)**:
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/user_management
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**:
```bash
# ml-service/.env
DATABASE_URL=mongodb://localhost:27017/user_management
MODEL_PATH=./models
LOG_LEVEL=INFO
```

**Frontend (.env)**:
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

## Key API Endpoints

### Authentication APIs

```javascript
// Backend route examples (Node.js/Express)
// backend/routes/auth.js

const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({
      name,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ token, user: { id: user._id, name, email, role: user.role } });
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
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ token, user: { id: user._id, name: user.name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Task Management APIs

```javascript
// backend/routes/tasks.js

const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authenticateToken } = require('../middleware/auth');

// Get all tasks (admin) or user tasks
router.get('/', authenticateToken, async (req, res) => {
  try {
    const query = req.user.role === 'admin' ? {} : { assignedTo: req.user.userId };
    const tasks = await Task.find(query).populate('assignedTo', 'name email');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching tasks', error: error.message });
  }
});

// Create task (admin only)
router.post('/', authenticateToken, async (req, res) => {
  try {
    if (req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Admin access required' });
    }
    
    const { title, description, assignedTo, priority, deadline } = req.body;
    const task = new Task({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      status: 'todo',
      deadline,
      createdBy: req.user.userId
    });
    
    await task.save();
    await task.populate('assignedTo', 'name email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error creating task', error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authenticateToken, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    if (req.user.role !== 'admin' && task.assignedTo.toString() !== req.user.userId) {
      return res.status(403).json({ message: 'Not authorized' });
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

module.exports = router;
```

### Support Ticket APIs

```javascript
// backend/routes/tickets.js

const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authenticateToken } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', authenticateToken, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for ticket classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
      title,
      description
    });
    
    const ticket = new Ticket({
      title,
      description,
      priority: priority || mlResponse.data.priority,
      category: mlResponse.data.category,
      assignedDepartment: mlResponse.data.department,
      createdBy: req.user.userId,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: 'Error creating ticket', error: error.message });
  }
});

// Get tickets
router.get('/', authenticateToken, async (req, res) => {
  try {
    const query = req.user.role === 'admin' ? {} : { createdBy: req.user.userId };
    const tickets = await Ticket.find(query).populate('createdBy', 'name email');
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching tickets', error: error.message });
  }
});

module.exports = router;
```

## ML Service (FastAPI)

### AI Analytics Endpoints

```python
# ml-service/main.py

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Models
risk_model = None
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_overdue: int
    avg_completion_time: float
    login_frequency: int

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    login_time: int  # hour of day
    location: str
    device: str
    failed_logins: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_hours_worked: float
    weekend_work: int

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """AI-based ticket classification and routing"""
    try:
        # Simple keyword-based classification (expand with ML model)
        text = (request.title + " " + request.description).lower()
        
        if "password" in text or "login" in text or "access" in text:
            category = "authentication"
            department = "IT Security"
            priority = "high"
        elif "bug" in text or "error" in text or "crash" in text:
            category = "technical"
            department = "Engineering"
            priority = "high"
        elif "feature" in text or "request" in text:
            category = "feature_request"
            department = "Product"
            priority = "medium"
        else:
            category = "general"
            department = "Support"
            priority = "low"
        
        return {
            "category": category,
            "department": department,
            "priority": priority
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavior patterns"""
    try:
        # Calculate risk score
        risk_score = 0
        
        # High overdue tasks increases risk
        if request.tasks_overdue > 3:
            risk_score += 30
        elif request.tasks_overdue > 0:
            risk_score += 15
        
        # Low completion rate increases risk
        if request.tasks_completed > 0:
            completion_rate = request.tasks_completed / (request.tasks_completed + request.tasks_overdue)
            if completion_rate < 0.5:
                risk_score += 25
        
        # Long completion time increases risk
        if request.avg_completion_time > 72:  # hours
            risk_score += 20
        
        # Low login frequency might indicate disengagement
        if request.login_frequency < 3:
            risk_score += 15
        
        risk_level = "high" if risk_score > 60 else "medium" if risk_score > 30 else "low"
        
        return {
            "user_id": request.user_id,
            "risk_score": min(risk_score, 100),
            "risk_level": risk_level,
            "factors": {
                "overdue_tasks": request.tasks_overdue,
                "completion_rate": request.tasks_completed / max(request.tasks_completed + request.tasks_overdue, 1),
                "avg_completion_time": request.avg_completion_time
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    try:
        # Create feature vector
        features = {
            'login_time': request.login_time,
            'failed_logins': request.failed_logins,
            'location_hash': hash(request.location) % 1000,
            'device_hash': hash(request.device) % 1000
        }
        
        # Score with anomaly detector
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomalous = score > 0.6
        
        anomaly_reasons = []
        if request.failed_logins > 3:
            anomaly_reasons.append("Multiple failed login attempts")
        if request.login_time < 6 or request.login_time > 22:
            anomaly_reasons.append("Unusual login time")
        
        return {
            "user_id": request.user_id,
            "is_anomalous": is_anomalous,
            "anomaly_score": float(score),
            "reasons": anomaly_reasons
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze user burnout risk"""
    try:
        burnout_score = 0
        
        # High workload
        if request.tasks_assigned > 15:
            burnout_score += 30
        
        # Low completion rate might indicate overload
        completion_rate = request.tasks_completed / max(request.tasks_assigned, 1)
        if completion_rate < 0.6:
            burnout_score += 25
        
        # Excessive hours
        if request.avg_hours_worked > 50:
            burnout_score += 35
        elif request.avg_hours_worked > 40:
            burnout_score += 20
        
        # Weekend work
        if request.weekend_work > 2:
            burnout_score += 20
        
        burnout_level = "high" if burnout_score > 60 else "medium" if burnout_score > 30 else "low"
        
        return {
            "user_id": request.user_id,
            "burnout_score": min(burnout_score, 100),
            "burnout_level": burnout_level,
            "recommendations": [
                "Reduce task load" if request.tasks_assigned > 15 else None,
                "Limit working hours" if request.avg_hours_worked > 45 else None,
                "Avoid weekend work" if request.weekend_work > 1 else None
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Frontend Integration

### React Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js

import { useState, useEffect, createContext, useContext } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUserProfile();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUserProfile = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/auth/profile`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    setUser(user);
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js

import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks`);
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
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
      await axios.patch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        <span className="deadline">{new Date(task.deadline).toLocaleDateString()}</span>
      </div>
    </div>
  );

  if (loading) return <div>Loading...</div>;

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do ({tasks.todo.length})</h3>
        <div className="task-list">
          {tasks.todo.map(task => (
            <TaskCard key={task._id} task={task} />
          ))}
        </div>
      </div>
      <div className="kanban-column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        <div className="task-list">
          {tasks.inProgress.map(task => (
            <TaskCard key={task._id} task={task} />
          ))}
        </div>
      </div>
      <div className="kanban-column">
        <h3>Done ({tasks.done.length})</h3>
        <div className="task-list">
          {tasks.done.map(task => (
            <TaskCard key={task._id} task={task} />
          ))}
        </div>
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard Component

```javascript
// frontend/src/components/AIAnalyticsDashboard.js

import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { Chart as ChartJS, ArcElement, Tooltip, Legend } from 'chart.js';
import { Pie } from 'react-chartjs-2';

ChartJS.register(ArcElement, Tooltip, Legend);

const AIAnalyticsDashboard = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    anomalies: [],
    burnoutRisk: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch user data and run AI analysis
      const usersResponse = await axios.get(`${process.env.REACT_APP_API_URL}/users`);
      const tasksResponse = await axios.get(`${process.env.REACT_APP_API_URL}/tasks`);
      
      // Analyze each user
      const riskAnalysis = await Promise.all(
        usersResponse.data.map(async (user) => {
          const userTasks = tasksResponse.data.filter(t => t.assignedTo._id === user._id);
          const completed = userTasks.filter(t => t.status === 'done').length;
          const overdue = userTasks.filter(t => new Date(t.deadline) < new Date() && t.status !== 'done').length;
          
          const riskResponse = await axios.post(`${process.env.REACT_APP_ML_URL}/predict-risk`, {
            user_id: user._id,
            tasks_completed: completed,
            tasks_overdue: overdue,
            avg_completion_time: 48,
            login_frequency: 5
          });
          
          return { user, risk: riskResponse.data };
        })
      );
      
      setAnalytics({
        riskUsers: riskAnalysis.filter(r => r.risk.risk_level === 'high'),
        anomalies: [],
        burnoutRisk: riskAnalysis.filter(r => r.risk.risk_score > 50)
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI Analytics Dashboard</h2>
      
      <div className="analytics-grid">
        <div className="analytics-card">
          <h3>High Risk Users</h3>
          <div className="risk-list">
            {analytics.riskUsers.map(({ user, risk }) => (
              <div key={user._id} className="risk-item">
                <span>{user.name}</span>
                <span className="risk-score">{risk.risk_score}%</span>
              </div>
            ))}
          </div>
        </div>
        
        <div className="analytics-card">
          <h3>Burnout Risk</h3>
          <div className="burnout-list">
            {analytics.burnoutRisk.map(({ user, risk }) => (
              <div key={user._id} className="burnout-item">
                <span>{user.name}</span>
                <span className={`level ${risk.risk_level}`}>{risk.risk_level}</span>
              </div>
            ))}
          </div>
        </div>
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Database Models

### MongoDB Schemas

```javascript
// backend/models/User.js

const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['admin', 'user'],
    default: 'user'
  },
  department: String,
  isActive: {
    type: Boolean,
    default: true
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: Date
});

module.exports = mongoose.model('User', userSchema);
```

```javascript
// backend/models/Task.js

const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  status: {
    type: String,
    enum: ['todo', 'in_progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  deadline: Date,
  timeTracked: {
    type: Number,
    default: 0  // in minutes
  },
  completedAt: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', taskSchema);
```

## Common Patterns

### Middleware for Authentication

```javascript
// backend/middleware/auth.js

const jwt = require('jsonwebtoken');

const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ message: 'Access token required' });
  }
  
  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) {
      return res.status(403).json({ message: 'Invalid or expired token' });
    }
    req.user = user;
    next();
  });
};

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authenticateToken, requireAdmin };
```

### Time Tracking Implementation

```javascript
// frontend/src/components/TimeTracker.js

import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const handleStart = () => setIsRunning(true);
  
  const handleStop = async () => {
    setIsRunning(false);
    try {
      await axios.patch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`, {
        timeAdded: Math.floor(seconds / 60)  // Convert to minutes
      });
    } catch (error) {
      console.error('Error saving time:', error);
    }
  };

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={handleStart} disabled={isRunning}>Start</button>
      <button onClick={handleStop} disabled={!isRunning}>Stop</button>
    </div>
  );
};

export default TimeTracker;
```

## Troubleshooting

### Connection Issues

**MongoDB connection errors:**
```bash
# Check MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Check connection in backend
# Ensure MONGODB_URI in .env is correct
```

**CORS errors:**
```javascript
// backend/server.js - Add CORS configuration

const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Authentication Issues

**JWT token expired:**
```javascript
// Frontend axios interceptor to handle token refresh

axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response.status === 403) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### ML Service Issues

**Model not loading:**
```python
# Ensure models directory exists
import os
os.makedirs('./models', exist_ok=True)

# Check Python dependencies
pip install -r requirements.txt --upgrade
```

**Prediction errors:**
```python
# Add error handling and validation
from pydantic import validator

class RiskPredictionRequest(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_overdue: int
    
    @validator('tasks_completed', 'tasks_overdue')
    def check_non_negative(cls, v):
        if v
