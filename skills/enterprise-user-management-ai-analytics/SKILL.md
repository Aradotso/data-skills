---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management with AI"
  - "implement user task tracking with AI analytics"
  - "create admin dashboard with anomaly detection"
  - "build ticket management system with ML classification"
  - "add AI risk prediction to user management"
  - "integrate burnout detection for employees"
  - "develop kanban board with time tracking"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management AI Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines traditional user/task management with ML-powered insights. It provides role-based access control, Kanban task boards, support ticket management, and AI features including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

**Stack:** React frontend, Node.js/Express backend, FastAPI ML service, MongoDB database, JWT authentication.

## Installation

### Prerequisites

```bash
# Node.js 14+ and Python 3.8+ required
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env for ML service
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Key Architecture

### Backend API Structure

The Node.js backend exposes REST APIs organized by feature:

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/tasks', require('./routes/tasks'));
app.use('/api/tickets', require('./routes/tickets'));
app.use('/api/analytics', require('./routes/analytics'));

// Connect to MongoDB
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
});

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### Authentication System

JWT-based authentication with role-based access:

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization').replace('Bearer ', '');
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findOne({ _id: decoded.id });

    if (!user) {
      throw new Error();
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).send({ error: 'Please authenticate' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).send({ error: 'Admin access required' });
  }
  next();
};

module.exports = { auth, adminOnly };
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
  position: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

// Method to check password
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model with Kanban States

```javascript
// backend/models/Task.js
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
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'urgent'], 
    default: 'medium' 
  },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in minutes
  tags: [String],
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
  category: { 
    type: String, 
    enum: ['technical', 'hr', 'access', 'bug', 'feature', 'other'],
    default: 'other'
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'urgent'] 
  },
  status: { 
    type: String, 
    enum: ['open', 'inprogress', 'resolved', 'closed'],
    default: 'open'
  },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassified: { type: Boolean, default: false },
  aiConfidence: Number,
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Backend API Endpoints

### Authentication Routes

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const User = require('../models/User');
const router = express.Router();

// Register
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'Email already exists' });
    }

    const user = new User({ name, email, password, role });
    await user.save();

    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });

    res.status(201).json({ user: { id: user._id, name, email, role }, token });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });

    if (!user || !(await user.comparePassword(password))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    user.lastLogin = new Date();
    await user.save();

    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });

    res.json({ 
      user: { 
        id: user._id, 
        name: user.name, 
        email: user.email, 
        role: user.role 
      }, 
      token 
    });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { auth } = require('../middleware/auth');
const router = express.Router();

// Get all tasks (user sees own, admin sees all)
router.get('/', auth, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user._id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', auth, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id
    });
    await task.save();
    await task.populate('assignedTo createdBy', 'name email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update task status (Kanban drag-drop)
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.status = status;
    if (status === 'done' && !task.completedAt) {
      task.completedAt = new Date();
    }

    await task.save();
    await task.populate('assignedTo createdBy', 'name email');
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update time spent
router.patch('/:id/time', auth, async (req, res) => {
  try {
    const { timeSpent } = req.body; // in minutes
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeSpent } },
      { new: true }
    ).populate('assignedTo createdBy', 'name email');

    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### Ticket Routes with AI Integration

```javascript
// backend/routes/tickets.js
const express = require('express');
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { auth } = require('../middleware/auth');
const router = express.Router();

// Create ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Call ML service for classification
    let aiData = {};
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { title, description }
      );
      aiData = {
        category: mlResponse.data.category,
        priority: mlResponse.data.priority,
        aiClassified: true,
        aiConfidence: mlResponse.data.confidence
      };
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
      // Fallback to default if ML service fails
    }

    const ticket = new Ticket({
      ...req.body,
      ...aiData,
      createdBy: req.user._id
    });

    await ticket.save();
    await ticket.populate('createdBy assignedTo', 'name email');
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get tickets
router.get('/', auth, async (req, res) => {
  try {
    const query = req.user.role === 'admin'
      ? {}
      : { createdBy: req.user._id };

    const tickets = await Ticket.find(query)
      .populate('createdBy assignedTo', 'name email')
      .sort({ createdAt: -1 });

    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service (FastAPI)

### ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Dict
import os

app = FastAPI(title="Enterprise AI Analytics Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_logins: int
    unusual_hours_access: int
    data_access_frequency: int
    location_changes: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    avg_task_time: float
    overtime_hours: float
    days_without_break: int

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """
    AI-powered ticket classification and priority assignment
    """
    text = f"{request.title} {request.description}".lower()
    
    # Simple keyword-based classification (replace with actual ML model)
    category_keywords = {
        'technical': ['bug', 'error', 'crash', 'not working', 'broken'],
        'hr': ['leave', 'payroll', 'benefits', 'holiday', 'salary'],
        'access': ['login', 'password', 'permission', 'access denied', 'unlock'],
        'feature': ['feature', 'enhancement', 'new', 'add', 'improve'],
        'bug': ['bug', 'issue', 'problem', 'error']
    }
    
    priority_keywords = {
        'urgent': ['critical', 'urgent', 'asap', 'emergency', 'down'],
        'high': ['important', 'high', 'blocking', 'severe'],
        'medium': ['medium', 'moderate', 'normal'],
        'low': ['minor', 'low', 'cosmetic', 'trivial']
    }
    
    # Find category
    category = 'other'
    max_score = 0
    for cat, keywords in category_keywords.items():
        score = sum(1 for kw in keywords if kw in text)
        if score > max_score:
            max_score = score
            category = cat
    
    # Find priority
    priority = 'medium'
    for pri, keywords in priority_keywords.items():
        if any(kw in text for kw in keywords):
            priority = pri
            break
    
    confidence = min(0.95, max(0.5, max_score * 0.2 + 0.5))
    
    return {
        "category": category,
        "priority": priority,
        "confidence": confidence
    }

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """
    Predict security risk based on user behavior
    """
    # Simple risk scoring algorithm
    risk_score = (
        request.failed_logins * 15 +
        request.unusual_hours_access * 10 +
        request.data_access_frequency * 5 +
        request.location_changes * 8
    )
    
    risk_score = min(100, risk_score)
    
    if risk_score >= 70:
        risk_level = "high"
        recommendation = "Immediate investigation required. Suspend access temporarily."
    elif risk_score >= 40:
        risk_level = "medium"
        recommendation = "Monitor closely. Review access logs."
    else:
        risk_level = "low"
        recommendation = "Normal behavior. Continue monitoring."
    
    return {
        "user_id": request.user_id,
        "risk_score": risk_score,
        "risk_level": risk_level,
        "recommendation": recommendation,
        "factors": {
            "failed_logins": request.failed_logins,
            "unusual_hours": request.unusual_hours_access,
            "data_access": request.data_access_frequency,
            "location_changes": request.location_changes
        }
    }

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """
    Analyze employee burnout risk
    """
    # Calculate burnout score
    burnout_score = (
        (request.avg_task_time / 60) * 10 +  # Normalize to hours
        request.overtime_hours * 2 +
        request.days_without_break * 5
    )
    
    workload_intensity = request.tasks_completed * (request.avg_task_time / 60)
    
    burnout_score = min(100, burnout_score)
    
    if burnout_score >= 70:
        burnout_level = "high"
        recommendation = "Immediate intervention needed. Reduce workload and mandate time off."
    elif burnout_score >= 40:
        burnout_level = "medium"
        recommendation = "Monitor closely. Consider workload redistribution."
    else:
        burnout_level = "low"
        recommendation = "Healthy work-life balance. Continue current pace."
    
    return {
        "user_id": request.user_id,
        "burnout_score": burnout_score,
        "burnout_level": burnout_level,
        "workload_intensity": round(workload_intensity, 2),
        "recommendation": recommendation,
        "metrics": {
            "tasks_completed": request.tasks_completed,
            "avg_task_time_hours": round(request.avg_task_time / 60, 2),
            "overtime_hours": request.overtime_hours,
            "days_without_break": request.days_without_break
        }
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "Enterprise AI Analytics"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### ML Service Requirements

```txt
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
numpy==1.24.3
scikit-learn==1.3.2
joblib==1.3.2
python-dotenv==1.0.0
river==0.19.0
```

## Frontend React Components

### API Client Setup

```javascript
// frontend/src/api/client.js
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL || 'http://localhost:5000',
  headers: {
    'Content-Type': 'application/json'
  }
});

const mlClient = axios.create({
  baseURL: process.env.REACT_APP_ML_API_URL || 'http://localhost:8000',
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add auth token to requests
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export { apiClient, mlClient };
```

### Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import { apiClient } from '../api/client';

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
    const { user, token } = response.data;
    
    localStorage.setItem('token', token);
    localStorage.setItem('user', JSON.stringify(user));
    setUser(user);
    
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
    setUser(null);
  };

  const isAdmin = user?.role === 'admin';

  return { user, login, logout, loading, isAdmin };
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import { apiClient } from '../api/client';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await apiClient.get('/api/tasks');
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status].push(task);
        return acc;
      }, { todo: [], inprogress: [], done: [] });
      
      setTasks(grouped);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    
    try {
      await apiClient.patch(`/api/tasks/${taskId}/status`, { status: newStatus });
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const columns = [
    { id: 'todo', title: 'To Do', color: '#e74c3c' },
    { id: 'inprogress', title: 'In Progress', color: '#f39c12' },
    { id: 'done', title: 'Done', color: '#27ae60' }
  ];

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {columns.map(column => (
        <div 
          key={column.id}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, column.id)}
          onDragOver={handleDragOver}
        >
          <h3 style={{ borderTop: `4px solid ${column.color}` }}>
            {column.title} ({tasks[column.id].length})
          </h3>
          <div className="task-list">
            {tasks[column.id].map(task => (
              <div
                key={task._id}
                className="task-card"
                draggable
                onDragStart={(e) => handleDragStart(e, task._id)}
              >
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <div className="task-meta">
                  <span className={`priority ${task.priority}`}>
                    {task.priority}
                  </span>
                  {task.assignedTo && (
                    <span className="assignee">
                      👤 {task.assignedTo.name}
                    </span>
                  )}
                </div>
                {task.timeSpent > 0 && (
                  <div className="time-spent">
                    ⏱️ {Math.floor(task.timeSpent / 60)}h {task.timeSpent % 60}m
                  </div>
                )}
              </div>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Time Tracker Component

```javascript
// frontend/src/components/TimeTracker.js
import React, { useState, useEffect } from 'react';
import { apiClient } from '../api/client';

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
    
    // Save time to backend
    if (seconds > 0 && taskId) {
      try {
        const minutes = Math.floor(seconds / 60);
        await apiClient.patch(`/api/tasks/${taskId}/time`, { 
          timeSpent: minutes 
        });
      } catch (error) {
        console.error('Error saving time:', error);
      }
    }
  };

  const handleReset = () => {
    setSeconds(0);
    setIsRunning(false);
  };

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <div className="timer-controls">
        {!isRunning ? (
          <button onClick={handleStart}>▶ Start</button>
        ) : (
          <button onClick={handleStop}>⏸ Stop</button>
        )}
        <button onClick={handleReset}>⏹ Reset</button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

### AI Risk Dashboard Component

```javascript
// frontend/src/components/RiskDashboard.js
import React, { useState, useEffect } from 'react';
import { mlClient } from '../api/client';

const RiskDashboard = ({ userId }) => {
  const [riskData, setRiskData] = useState(null);
  const [loading, setLoading] = useState(false);

  const analyzeRisk = async () => {
    setLoading(true);
    try {
      const response = await mlClient.post('/predict-risk', {
        user_id: userId,
        failed_logins: 3,
        unusual_hours_access: 5,
        data_access_frequency: 12,
        location_changes: 2
      });
      setRiskData(response.data);
    } catch (error) {
      console.error('Risk analysis error:', error);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    if (userId) {
      analyzeRisk();
    }
  }, [userId]);

  if (loading) return <div>Analyzing risk...</div>;
  if (!riskData) return null;

  const getRiskColor = (level) => {
    switch (level) {
      case 'high': return '#e74c3c';
      case 'medium': return '#f39c12';
      case 'low': return '#27ae60';
      default: return '#95a5a6';
    }
  };

  return (
    <div className="risk-dashboard">
      <h3>Security Risk Analysis</h3>
      <div 
        className="risk-score"
        style={{ backgroundColor: getRiskColor(riskData.risk_level) }}
      >
        <div className="score-value">{riskData.risk_score}/100</div>
        <div className="score-label">{riskData.risk_level.toUpperCase()} RISK</div>
      </div>
      
      <div className="risk-factors">
        <h4>Risk Factors:</h4>
        <ul>
