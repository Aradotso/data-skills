---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task management, ticket classification, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "create user management dashboard with AI analytics"
  - "implement AI ticket classification and routing"
  - "build task management with burnout detection"
  - "add risk prediction to user management"
  - "configure user management with ML insights"
  - "integrate AI analytics for enterprise users"
  - "deploy user management system with predictive analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system that combines traditional CRUD operations with AI-powered analytics including ticket classification, risk prediction, anomaly detection, burnout analysis, and project delay prediction. Built with React frontend, Node.js backend, MongoDB database, and FastAPI ML service using scikit-learn and River for online learning.

## Installation

### Prerequisites

Ensure you have Node.js 14+, Python 3.8+, and MongoDB installed.

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

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-users
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
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

Create `.env` file in ml-service directory:

```env
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

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Key Architecture

### Backend API Structure (Node.js)

The backend provides REST APIs for user management, authentication, tasks, and tickets:

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();

app.use(cors());
app.use(express.json());

// Connect to MongoDB
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
});

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/tasks', require('./routes/tasks'));
app.use('/api/tickets', require('./routes/tickets'));
app.use('/api/analytics', require('./routes/analytics'));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  tasks: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Task' }],
  performance: {
    tasksCompleted: { type: Number, default: 0 },
    averageCompletionTime: { type: Number, default: 0 },
    riskScore: { type: Number, default: 0 }
  },
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date
});

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
  status: { 
    type: String, 
    enum: ['todo', 'inprogress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  dueDate: Date,
  timeTracked: { type: Number, default: 0 }, // in seconds
  createdAt: { type: Date, default: Date.now },
  completedAt: Date
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
  category: String,
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: {
    category: String,
    confidence: Number,
    suggestedDepartment: String
  },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: `User role ${req.user.role} is not authorized to access this route` 
      });
    }
    next();
  };
};
```

### User Routes

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { protect, authorize } = require('../middleware/auth');

// Get all users (Admin only)
router.get('/', protect, authorize('admin'), async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user profile
router.get('/profile', protect, async (req, res) => {
  try {
    const user = await User.findById(req.user.id)
      .select('-password')
      .populate('tasks');
    res.json(user);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create user (Admin only)
router.post('/', protect, authorize('admin'), async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;
    
    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const user = await User.create({
      name,
      email,
      password,
      role,
      department
    });
    
    res.status(201).json({
      _id: user._id,
      name: user.name,
      email: user.email,
      role: user.role
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update user
router.put('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Delete user
router.delete('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json({ message: 'User removed' });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Task Routes with ML Integration

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const User = require('../models/User');
const axios = require('axios');
const { protect } = require('../middleware/auth');

// Get all tasks for user
router.get('/', protect, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task
router.post('/', protect, async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      assignedTo: req.body.assignedTo || req.user.id
    });
    
    await User.findByIdAndUpdate(
      task.assignedTo,
      { $push: { tasks: task._id } }
    );
    
    // Check for burnout risk
    const userTasks = await Task.find({ 
      assignedTo: task.assignedTo,
      status: { $ne: 'done' }
    });
    
    if (userTasks.length >= 5) {
      try {
        const mlResponse = await axios.post(
          `${process.env.ML_SERVICE_URL}/predict/burnout`,
          {
            userId: task.assignedTo.toString(),
            activeTasks: userTasks.length,
            averageTimePerTask: userTasks.reduce((sum, t) => sum + t.timeTracked, 0) / userTasks.length
          }
        );
        
        if (mlResponse.data.burnoutRisk > 0.7) {
          console.log(`High burnout risk detected for user ${task.assignedTo}`);
        }
      } catch (mlError) {
        console.error('ML service error:', mlError.message);
      }
    }
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.status = status;
    
    if (status === 'done') {
      task.completedAt = new Date();
      
      // Update user performance
      const user = await User.findById(task.assignedTo);
      user.performance.tasksCompleted += 1;
      await user.save();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update time tracked
router.patch('/:id/time', protect, async (req, res) => {
  try {
    const { timeTracked } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeTracked } },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service with FastAPI

### Main ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, List
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
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

# Models directory
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# In-memory models (in production, load from disk)
ticket_classifier = None
risk_predictor = None
scaler = StandardScaler()

class TicketRequest(BaseModel):
    title: str
    description: str
    priority: str

class TicketClassification(BaseModel):
    category: str
    confidence: float
    suggestedDepartment: str

class RiskRequest(BaseModel):
    userId: str
    taskCompletionRate: float
    averageResponseTime: float
    failedLogins: int
    lastLoginHours: float

class RiskPrediction(BaseModel):
    riskScore: float
    riskLevel: str
    factors: List[str]

class BurnoutRequest(BaseModel):
    userId: str
    activeTasks: int
    averageTimePerTask: float
    workHoursPerWeek: Optional[float] = 40.0

class BurnoutPrediction(BaseModel):
    burnoutRisk: float
    recommendation: str

@app.get("/")
def read_root():
    return {"message": "Enterprise User Management ML Service"}

@app.post("/classify/ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketRequest):
    """
    Classify support ticket into category and suggest department
    """
    try:
        # Simple rule-based classification (replace with trained model)
        text = f"{ticket.title} {ticket.description}".lower()
        
        categories = {
            'technical': ['bug', 'error', 'crash', 'broken', 'not working', 'issue'],
            'access': ['login', 'password', 'access', 'permission', 'denied'],
            'feature': ['request', 'feature', 'enhancement', 'suggestion'],
            'account': ['account', 'profile', 'settings', 'email'],
            'general': []
        }
        
        department_mapping = {
            'technical': 'Engineering',
            'access': 'IT Support',
            'feature': 'Product',
            'account': 'Customer Success',
            'general': 'Support'
        }
        
        category = 'general'
        confidence = 0.5
        
        for cat, keywords in categories.items():
            matches = sum(1 for keyword in keywords if keyword in text)
            if matches > 0:
                category = cat
                confidence = min(0.95, 0.6 + (matches * 0.1))
                break
        
        return TicketClassification(
            category=category,
            confidence=confidence,
            suggestedDepartment=department_mapping[category]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict/risk", response_model=RiskPrediction)
async def predict_risk(request: RiskRequest):
    """
    Predict user risk score based on behavior patterns
    """
    try:
        # Calculate risk score based on multiple factors
        factors = []
        risk_score = 0.0
        
        # Task completion rate factor
        if request.taskCompletionRate < 0.5:
            risk_score += 0.3
            factors.append("Low task completion rate")
        
        # Response time factor
        if request.averageResponseTime > 48:  # hours
            risk_score += 0.2
            factors.append("Slow response time")
        
        # Failed logins factor
        if request.failedLogins > 3:
            risk_score += 0.25
            factors.append("Multiple failed login attempts")
        
        # Activity factor
        if request.lastLoginHours > 168:  # 1 week
            risk_score += 0.25
            factors.append("Inactive user")
        
        risk_score = min(1.0, risk_score)
        
        if risk_score < 0.3:
            risk_level = "Low"
        elif risk_score < 0.6:
            risk_level = "Medium"
        else:
            risk_level = "High"
        
        return RiskPrediction(
            riskScore=round(risk_score, 2),
            riskLevel=risk_level,
            factors=factors if factors else ["No risk factors detected"]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict/burnout", response_model=BurnoutPrediction)
async def predict_burnout(request: BurnoutRequest):
    """
    Predict employee burnout risk based on workload
    """
    try:
        # Calculate burnout risk
        burnout_risk = 0.0
        
        # Active tasks factor (threshold: 8 tasks)
        if request.activeTasks > 8:
            burnout_risk += 0.4
        elif request.activeTasks > 5:
            burnout_risk += 0.2
        
        # Time per task factor (threshold: 4 hours average)
        if request.averageTimePerTask > 14400:  # seconds
            burnout_risk += 0.3
        elif request.averageTimePerTask > 10800:
            burnout_risk += 0.15
        
        # Work hours factor
        if request.workHoursPerWeek > 50:
            burnout_risk += 0.3
        elif request.workHoursPerWeek > 45:
            burnout_risk += 0.15
        
        burnout_risk = min(1.0, burnout_risk)
        
        if burnout_risk < 0.3:
            recommendation = "Workload is manageable"
        elif burnout_risk < 0.6:
            recommendation = "Consider redistributing tasks"
        else:
            recommendation = "High burnout risk - immediate action needed"
        
        return BurnoutPrediction(
            burnoutRisk=round(burnout_risk, 2),
            recommendation=recommendation
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect/anomaly")
async def detect_anomaly(data: dict):
    """
    Detect anomalies in user behavior
    """
    try:
        # Simple threshold-based anomaly detection
        anomalies = []
        
        if data.get('loginAttempts', 0) > 10:
            anomalies.append("Unusual number of login attempts")
        
        if data.get('dataAccessed', 0) > 1000:
            anomalies.append("Excessive data access")
        
        if data.get('loginHour', 12) < 6 or data.get('loginHour', 12) > 22:
            anomalies.append("Login at unusual hours")
        
        return {
            "isAnomaly": len(anomalies) > 0,
            "anomalyScore": min(1.0, len(anomalies) * 0.4),
            "anomalies": anomalies
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Requirements File

```python
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
scikit-learn==1.3.2
numpy==1.24.3
joblib==1.3.2
python-dotenv==1.0.0
river==0.19.0
```

## Frontend React Components

### API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to requests
api.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authService = {
  login: async (email, password) => {
    const response = await api.post('/api/auth/login', { email, password });
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
    }
    return response.data;
  },
  
  register: async (userData) => {
    const response = await api.post('/api/auth/register', userData);
    return response.data;
  },
  
  logout: () => {
    localStorage.removeItem('token');
  }
};

export const userService = {
  getProfile: () => api.get('/api/users/profile'),
  getAllUsers: () => api.get('/api/users'),
  createUser: (userData) => api.post('/api/users', userData),
  updateUser: (id, userData) => api.put(`/api/users/${id}`, userData),
  deleteUser: (id) => api.delete(`/api/users/${id}`)
};

export const taskService = {
  getTasks: () => api.get('/api/tasks'),
  createTask: (taskData) => api.post('/api/tasks', taskData),
  updateTaskStatus: (id, status) => api.patch(`/api/tasks/${id}/status`, { status }),
  updateTimeTracked: (id, timeTracked) => api.patch(`/api/tasks/${id}/time`, { timeTracked })
};

export const ticketService = {
  getTickets: () => api.get('/api/tickets'),
  createTicket: async (ticketData) => {
    // Get AI classification first
    const mlResponse = await axios.post(`${ML_URL}/classify/ticket`, {
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority
    });
    
    return api.post('/api/tickets', {
      ...ticketData,
      aiClassification: mlResponse.data
    });
  },
  updateTicket: (id, ticketData) => api.patch(`/api/tickets/${id}`, ticketData)
};

export const mlService = {
  classifyTicket: (ticketData) => axios.post(`${ML_URL}/classify/ticket`, ticketData),
  predictRisk: (userData) => axios.post(`${ML_URL}/predict/risk`, userData),
  predictBurnout: (workloadData) => axios.post(`${ML_URL}/predict/burnout`, workloadData),
  detectAnomaly: (behaviorData) => axios.post(`${ML_URL}/detect/anomaly`, behaviorData)
};

export default api;
```

### Task Board Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';
import './TaskBoard.css';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    inprogress: [],
    done: []
  });
  const [activeTimer, setActiveTimer] = useState(null);
  const [elapsedTime, setElapsedTime] = useState(0);

  useEffect(() => {
    loadTasks();
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

  const loadTasks = async () => {
    try {
      const response = await taskService.getTasks();
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inprogress: response.data.filter(t => t.status === 'inprogress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error loading tasks:', error);
    }
  };

  const handleDragStart = (e, taskId, status) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('sourceStatus', status);
  };

  const handleDrop = async (e, targetStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const sourceStatus = e.dataTransfer.getData('sourceStatus');

    if (sourceStatus === targetStatus) return;

    try {
      await taskService.updateTaskStatus(taskId, targetStatus);
      loadTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const startTimer = (taskId) => {
    setActiveTimer(taskId);
    setElapsedTime(0);
  };

  const stopTimer = async (taskId) => {
    if (activeTimer === taskId) {
      try {
        await taskService.updateTimeTracked(taskId, elapsedTime);
        setActiveTimer(null);
        setElapsedTime(0);
        loadTasks();
      } catch (error) {
        console.error('Error updating time:', error);
      }
    }
  };

  const formatTime = (seconds) => {
    const hours = Math.floor(seconds / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const renderTask = (task, status) => (
    <div
      key={task._id}
      className="task-card"
      draggable
      onDragStart={(e) => handleDragStart(e, task._id, status)}
    >
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority priority-${task.priority}`}>{task.priority}</span>
        <span className="time-tracked">{formatTime(task.timeTracked)}</span>
      </div>
      <div className="timer-controls">
        {activeTimer === task._id ? (
          <>
            <span className="timer-display">{formatTime(elapsedTime)}</span>
            <button onClick={() => stopTimer(task._id)} className="btn-stop">Stop</button>
          </>
        ) : (
          <button onClick={() => startTimer(task._id)} className="btn-start">Start Timer</button>
        )}
      </div>
    </div>
  );

  return (
    <div className="task-board">
      <div
        className="task-column"
        onDrop={(e) => handleDrop(e, 'todo')}
        onDragOver={handleDragOver}
      >
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => renderTask(task, 'todo'))}
      </div>
      
      <div
        className="task-column"
        onDrop={(e) => handleDrop(e, 'inprogress')}
        onDragOver={handleDragOver}
      >
        <h3>In Progress ({tasks.inprogress.length})</h3>
        {tasks.inprogress.map(task => renderTask(task, 'inprogress'))}
      </div>
      
      <div
        className="task-column"
        onDrop={(e) => handleDrop(e, 'done')}
        onDragOver={handleDragOver}
      >
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => renderTask(task, 'done'))}
