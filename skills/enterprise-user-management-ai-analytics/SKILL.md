---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "create user dashboard with task tracking"
  - "implement AI ticket classification system"
  - "add burnout detection and risk prediction"
  - "build admin panel for user management"
  - "configure AI-powered analytics dashboard"
  - "deploy user management system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform combining React frontend, Node.js backend, and FastAPI ML service for intelligent task management, ticket routing, and predictive analytics including risk detection, anomaly detection, and burnout analysis.

## What This Project Does

This system provides a comprehensive platform for:
- **User Management**: Role-based access control, user CRUD operations
- **Task Management**: Kanban board, time tracking, task assignment
- **Ticket System**: Support ticket creation and AI-powered classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, performance monitoring

## Installation

### Prerequisites
- Node.js 14+
- Python 3.8+
- MongoDB
- npm or yarn

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
MONGODB_URI=mongodb://localhost:27017/enterprise_management
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
MONGODB_URI=mongodb://localhost:27017/enterprise_management
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

Create `.env` file:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
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

// Register user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
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
      { expiresIn: '24h' }
    );
    
    res.status(201).json({ token, user: { id: user._id, username, email, role: user.role } });
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
    
    const isValid = await bcrypt.compare(password, user.password);
    if (!isValid) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### User Management Endpoints

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const authMiddleware = require('../middleware/auth');
const adminMiddleware = require('../middleware/admin');

// Get all users (admin only)
router.get('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user by ID
router.get('/:id', authMiddleware, async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update user
router.put('/:id', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const { username, email, role, status } = req.body;
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { username, email, role, status },
      { new: true }
    ).select('-password');
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Delete user
router.delete('/:id', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    await User.findByIdAndDelete(req.params.id);
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Management Endpoints

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const authMiddleware = require('../middleware/auth');

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority, dueDate, assignedTo } = req.body;
    
    const task = new Task({
      title,
      description,
      priority,
      dueDate,
      assignedTo,
      createdBy: req.user.userId,
      status: 'todo'
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tasks
router.get('/my-tasks', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username');
    res.json(tasks);
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
router.post('/:id/track-time', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body;
    const task = await Task.findById(req.params.id);
    
    task.timeTracked = (task.timeTracked || 0) + duration;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Ticket Management Endpoints

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const authMiddleware = require('../middleware/auth');
const axios = require('axios');

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
      title,
      description
    });
    
    const ticket = new Ticket({
      title,
      description,
      priority,
      category: mlResponse.data.category,
      assignedDepartment: mlResponse.data.department,
      createdBy: req.user.userId,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get tickets
router.get('/', authMiddleware, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.userId };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update ticket
router.patch('/:id', authMiddleware, async (req, res) => {
  try {
    const { status, resolution } = req.body;
    const ticket = await Ticket.findByIdAndUpdate(
      req.params.id,
      { status, resolution, updatedAt: Date.now() },
      { new: true }
    );
    res.json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
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
from typing import List, Dict, Optional
import joblib
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from river import anomaly, tree
import os

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
ticket_classifier = None
vectorizer = None
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)

# Load or initialize models
def load_models():
    global ticket_classifier, vectorizer
    model_path = os.getenv('MODEL_PATH', './models')
    
    try:
        ticket_classifier = joblib.load(f'{model_path}/ticket_classifier.pkl')
        vectorizer = joblib.load(f'{model_path}/vectorizer.pkl')
    except:
        # Initialize with default
        vectorizer = TfidfVectorizer(max_features=1000)
        ticket_classifier = MultinomialNB()

load_models()

class TicketInput(BaseModel):
    title: str
    description: str

class RiskInput(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    activityScore: float
    lastLoginHours: float

class BurnoutInput(BaseModel):
    userId: str
    tasksCompleted: int
    tasksOverdue: int
    avgWorkHours: float
    weekendWork: int

class AnomalyInput(BaseModel):
    features: Dict[str, float]

@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketInput):
    try:
        # Combine title and description
        text = f"{ticket.title} {ticket.description}"
        
        # Simple rule-based classification
        text_lower = text.lower()
        
        if any(word in text_lower for word in ['bug', 'error', 'crash', 'not working']):
            category = 'technical'
            department = 'IT Support'
        elif any(word in text_lower for word in ['account', 'login', 'password', 'access']):
            category = 'account'
            department = 'Account Management'
        elif any(word in text_lower for word in ['payment', 'billing', 'invoice']):
            category = 'billing'
            department = 'Finance'
        else:
            category = 'general'
            department = 'General Support'
        
        return {
            "category": category,
            "department": department,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(data: RiskInput):
    try:
        # Calculate risk score
        risk_score = 0
        
        # Failed login attempts
        if data.failedLogins > 3:
            risk_score += 30
        
        # Unusual login patterns
        if data.lastLoginHours > 24:
            risk_score += 20
        
        # Low activity
        if data.activityScore < 0.3:
            risk_score += 25
        
        # Excessive login attempts
        if data.loginAttempts > 10:
            risk_score += 25
        
        risk_level = "high" if risk_score > 60 else "medium" if risk_score > 30 else "low"
        
        return {
            "userId": data.userId,
            "riskScore": min(risk_score, 100),
            "riskLevel": risk_level,
            "factors": {
                "failedLogins": data.failedLogins,
                "unusualActivity": data.activityScore < 0.3,
                "excessiveAttempts": data.loginAttempts > 10
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(data: BurnoutInput):
    try:
        burnout_score = 0
        
        # Overdue tasks
        if data.tasksOverdue > 5:
            burnout_score += 30
        
        # Long work hours
        if data.avgWorkHours > 50:
            burnout_score += 35
        
        # Weekend work
        if data.weekendWork > 2:
            burnout_score += 20
        
        # Task completion ratio
        if data.tasksCompleted > 0:
            completion_ratio = data.tasksOverdue / data.tasksCompleted
            if completion_ratio > 0.5:
                burnout_score += 15
        
        risk_level = "high" if burnout_score > 60 else "medium" if burnout_score > 30 else "low"
        
        return {
            "userId": data.userId,
            "burnoutScore": min(burnout_score, 100),
            "riskLevel": risk_level,
            "recommendations": [
                "Reduce workload" if burnout_score > 60 else "Monitor progress",
                "Take breaks" if data.avgWorkHours > 50 else "Maintain balance",
                "Avoid weekend work" if data.weekendWork > 2 else "Good work-life balance"
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(data: AnomalyInput):
    try:
        # Convert features to array
        feature_vector = list(data.features.values())
        
        # Use River for online anomaly detection
        score = anomaly_detector.score_one(data.features)
        anomaly_detector.learn_one(data.features)
        
        is_anomaly = score > 0.7
        
        return {
            "isAnomaly": is_anomaly,
            "score": float(score),
            "severity": "high" if score > 0.8 else "medium" if score > 0.7 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-delay")
async def predict_delay(project_data: Dict):
    try:
        # Simple delay prediction based on progress
        total_tasks = project_data.get('totalTasks', 0)
        completed_tasks = project_data.get('completedTasks', 0)
        days_elapsed = project_data.get('daysElapsed', 0)
        deadline_days = project_data.get('deadlineDays', 0)
        
        if total_tasks == 0:
            return {"delayProbability": 0, "estimatedDelay": 0}
        
        completion_rate = completed_tasks / total_tasks
        time_rate = days_elapsed / deadline_days if deadline_days > 0 else 1
        
        # If time elapsed is greater than completion rate, likely delay
        delay_probability = max(0, min(100, (time_rate - completion_rate) * 100))
        
        estimated_delay = int((total_tasks / (completed_tasks / days_elapsed if completed_tasks > 0 else 1)) - deadline_days)
        estimated_delay = max(0, estimated_delay)
        
        return {
            "delayProbability": round(delay_probability, 2),
            "estimatedDelay": estimated_delay,
            "riskLevel": "high" if delay_probability > 60 else "medium" if delay_probability > 30 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Frontend React Components

### Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
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
    
    localStorage.setItem('token', response.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
    setUser(response.data.user);
    
    return response.data;
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
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks/my-tasks`);
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
      await axios.patch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task, onStatusChange }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        <span className="due-date">{new Date(task.dueDate).toLocaleDateString()}</span>
      </div>
      <select 
        value={task.status} 
        onChange={(e) => onStatusChange(task._id, e.target.value)}
      >
        <option value="todo">To Do</option>
        <option value="in-progress">In Progress</option>
        <option value="done">Done</option>
      </select>
    </div>
  );

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => (
          <TaskCard key={task._id} task={task} onStatusChange={updateTaskStatus} />
        ))}
      </div>
      <div className="kanban-column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => (
          <TaskCard key={task._id} task={task} onStatusChange={updateTaskStatus} />
        ))}
      </div>
      <div className="kanban-column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => (
          <TaskCard key={task._id} task={task} onStatusChange={updateTaskStatus} />
        ))}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './AIAnalytics.css';

const AIAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutScore: null,
    anomalies: [],
    projectDelay: null
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch risk prediction
      const riskResponse = await axios.post(`${process.env.REACT_APP_ML_API_URL}/predict-risk`, {
        userId: 'current-user-id',
        loginAttempts: 5,
        failedLogins: 1,
        activityScore: 0.75,
        lastLoginHours: 2
      });

      // Fetch burnout detection
      const burnoutResponse = await axios.post(`${process.env.REACT_APP_ML_API_URL}/detect-burnout`, {
        userId: 'current-user-id',
        tasksCompleted: 20,
        tasksOverdue: 3,
        avgWorkHours: 45,
        weekendWork: 1
      });

      setAnalytics({
        riskScore: riskResponse.data,
        burnoutScore: burnoutResponse.data,
        anomalies: [],
        projectDelay: null
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const RiskIndicator = ({ score, level }) => (
    <div className={`risk-indicator ${level}`}>
      <h4>Risk Level: {level.toUpperCase()}</h4>
      <div className="progress-bar">
        <div className="progress" style={{ width: `${score}%` }}></div>
      </div>
      <p>Score: {score}/100</p>
    </div>
  );

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI Analytics</h2>
      
      {analytics.riskScore && (
        <div className="analytics-card">
          <h3>Security Risk Assessment</h3>
          <RiskIndicator 
            score={analytics.riskScore.riskScore} 
            level={analytics.riskScore.riskLevel}
          />
          <div className="risk-factors">
            <p>Failed Logins: {analytics.riskScore.factors.failedLogins}</p>
            <p>Unusual Activity: {analytics.riskScore.factors.unusualActivity ? 'Yes' : 'No'}</p>
          </div>
        </div>
      )}

      {analytics.burnoutScore && (
        <div className="analytics-card">
          <h3>Burnout Detection</h3>
          <RiskIndicator 
            score={analytics.burnoutScore.burnoutScore} 
            level={analytics.burnoutScore.riskLevel}
          />
          <div className="recommendations">
            <h4>Recommendations:</h4>
            <ul>
              {analytics.burnoutScore.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Configuration

### Database Models

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  createdAt: { type: Date, default: Date.now },
  lastLogin: { type: Date }
});

module.exports = mongoose.model('User', userSchema);
```

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  dueDate: { type: Date },
  timeTracked: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

module.exports = (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return res.status(401).json({ error: 'No token provided' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};
```

```javascript
// backend/middleware/admin.js
module.exports = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};
```

## Common Patterns

### Creating a New User (Admin)

```javascript
const createUser = async (userData) => {
  try {
    const response = await axios.post(
      `${process.env.REACT_APP_API_URL}/users`,
      {
        username: userData.username,
        email: userData.email,
        password: userData.password,
        role: userData.role
      },
      {
        headers: {
          Authorization: `Bearer ${localStorage.getItem('token')}`
        }
      }
    );
    return response.data;
  } catch (error) {
    console.error('Error creating user:', error);
    throw error;
  }
};
```

### Submitting a Ticket with AI Classification

```javascript
const submitTicket = async (ticketData) => {
  try {
    const response = await axios.post(
      `${process.env.REACT_APP_API_URL}/tickets`,
      {
        title: ticketData.title,
        description: ticketData.description,
