---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and risk detection
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "build admin dashboard with task assignment"
  - "implement AI ticket classification and routing"
  - "create user management system with JWT authentication"
  - "add anomaly detection to user tracking"
  - "develop kanban board with time tracking"
  - "implement burnout detection for users"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user/task management with machine learning capabilities. The system provides:

- **User Management**: Role-based access control, JWT authentication, user CRUD operations
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization-wide analytics, audit logs, alerts

**Stack**: React frontend, Node.js/Express backend, FastAPI ML service, MongoDB database

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
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
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
# Runs on http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=${MONGODB_URI}
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
# Runs on http://localhost:8000
```

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
# Runs on http://localhost:3000
```

## Backend API Structure

### Authentication (Node.js/Express)

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

const router = express.Router();

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
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ success: false, error: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ success: false, error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({
      success: true,
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

### Middleware for Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const auth = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { auth, adminOnly };
```

### Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { auth, adminOnly } = require('../middleware/auth');

const router = express.Router();

// Create task (admin only)
router.post('/', auth, adminOnly, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      status: 'todo',
      dueDate,
      createdBy: req.user.id
    });
    
    await task.save();
    res.status(201).json({ success: true, task });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Get user tasks
router.get('/my-tasks', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, tasks });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    const { id } = req.params;
    
    const task = await Task.findById(id);
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    // Check authorization
    if (task.assignedTo.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Not authorized' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json({ success: true, task });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Track time on task
router.post('/:id/time', auth, async (req, res) => {
  try {
    const { duration } = req.body; // in seconds
    const { id } = req.params;
    
    const task = await Task.findById(id);
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    task.timeSpent = (task.timeSpent || 0) + duration;
    await task.save();
    
    res.json({ success: true, task });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

### Ticket Management API

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
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      { title, description }
    );
    
    const ticket = new Ticket({
      title,
      description,
      priority: priority || 'medium',
      category: mlResponse.data.category,
      suggestedAssignee: mlResponse.data.suggested_assignee,
      createdBy: req.user.id,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json({ success: true, ticket });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Get all tickets
router.get('/', auth, async (req, res) => {
  try {
    const query = req.user.role === 'admin' ? {} : { createdBy: req.user.id };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, tickets });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

## ML Service (FastAPI)

### Ticket Classification

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import joblib
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.ensemble import RandomForestClassifier

app = FastAPI()

# Load or initialize models
try:
    ticket_classifier = joblib.load('./models/ticket_classifier.pkl')
    vectorizer = joblib.load('./models/vectorizer.pkl')
except:
    vectorizer = TfidfVectorizer(max_features=100)
    ticket_classifier = RandomForestClassifier(n_estimators=100)

class TicketData(BaseModel):
    title: str
    description: str

class UserBehavior(BaseModel):
    user_id: str
    login_attempts: int
    failed_logins: int
    unusual_hours: bool
    data_access_volume: int

@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketData):
    try:
        text = f"{ticket.title} {ticket.description}"
        features = vectorizer.transform([text])
        
        category = ticket_classifier.predict(features)[0]
        confidence = np.max(ticket_classifier.predict_proba(features))
        
        # Simple routing logic
        assignee_map = {
            'technical': 'tech_team',
            'billing': 'finance_team',
            'general': 'support_team'
        }
        
        return {
            "category": category,
            "confidence": float(confidence),
            "suggested_assignee": assignee_map.get(category, 'support_team')
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(behavior: UserBehavior):
    try:
        # Simple rule-based anomaly detection
        risk_score = 0
        
        if behavior.failed_logins > 3:
            risk_score += 30
        if behavior.unusual_hours:
            risk_score += 20
        if behavior.data_access_volume > 1000:
            risk_score += 25
        if behavior.login_attempts > 10:
            risk_score += 25
        
        is_anomaly = risk_score > 50
        
        return {
            "is_anomaly": is_anomaly,
            "risk_score": risk_score,
            "risk_level": "high" if risk_score > 70 else "medium" if risk_score > 40 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-burnout")
async def predict_burnout(data: dict):
    try:
        tasks_count = data.get('tasks_count', 0)
        overtime_hours = data.get('overtime_hours', 0)
        days_since_break = data.get('days_since_break', 0)
        
        # Simple burnout scoring
        burnout_score = (
            (tasks_count / 20) * 40 +
            (overtime_hours / 20) * 30 +
            (days_since_break / 30) * 30
        )
        
        burnout_risk = "high" if burnout_score > 70 else "medium" if burnout_score > 40 else "low"
        
        return {
            "burnout_score": min(100, burnout_score),
            "risk_level": burnout_risk,
            "recommendations": [
                "Take a break" if days_since_break > 20 else None,
                "Reduce workload" if tasks_count > 15 else None,
                "Limit overtime" if overtime_hours > 15 else None
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Risk Prediction

```python
# ml-service/risk_prediction.py
from river import tree, ensemble
import json

class RiskPredictor:
    def __init__(self):
        # Online learning model
        self.model = ensemble.AdaptiveRandomForestClassifier(
            n_models=10,
            seed=42
        )
    
    def predict_risk(self, user_features):
        """
        Predict risk based on user behavior
        """
        features = {
            'login_frequency': user_features.get('login_frequency', 0),
            'failed_attempts': user_features.get('failed_attempts', 0),
            'data_downloads': user_features.get('data_downloads', 0),
            'weekend_activity': int(user_features.get('weekend_activity', False)),
            'vpn_usage': int(user_features.get('vpn_usage', False))
        }
        
        risk_prob = self.model.predict_proba_one(features)
        return {
            'risk_probability': risk_prob.get(1, 0) if risk_prob else 0.0,
            'is_risky': risk_prob.get(1, 0) > 0.7 if risk_prob else False
        }
    
    def update_model(self, features, is_risky):
        """
        Online learning update
        """
        self.model.learn_one(features, is_risky)
```

## Frontend (React)

### User Dashboard

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [stats, setStats] = useState({});
  const [loading, setLoading] = useState(true);
  
  const API_URL = process.env.REACT_APP_API_URL;
  
  useEffect(() => {
    fetchTasks();
    fetchStats();
  }, []);
  
  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(`${API_URL}/api/tasks/my-tasks`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      setTasks(response.data.tasks);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };
  
  const fetchStats = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(`${API_URL}/api/users/stats`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      setStats(response.data);
    } catch (error) {
      console.error('Error fetching stats:', error);
    }
  };
  
  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };
  
  const renderKanbanBoard = () => {
    const columns = {
      todo: tasks.filter(t => t.status === 'todo'),
      inProgress: tasks.filter(t => t.status === 'inProgress'),
      done: tasks.filter(t => t.status === 'done')
    };
    
    return (
      <div className="kanban-board">
        {Object.keys(columns).map(status => (
          <div key={status} className="kanban-column">
            <h3>{status.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
            {columns[status].map(task => (
              <div key={task._id} className="task-card">
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <div className="task-actions">
                  {status !== 'done' && (
                    <button onClick={() => updateTaskStatus(
                      task._id,
                      status === 'todo' ? 'inProgress' : 'done'
                    )}>
                      Move →
                    </button>
                  )}
                </div>
              </div>
            ))}
          </div>
        ))}
      </div>
    );
  };
  
  if (loading) return <div>Loading...</div>;
  
  return (
    <div className="user-dashboard">
      <h1>My Dashboard</h1>
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Tasks</h3>
          <p>{tasks.length}</p>
        </div>
        <div className="stat-card">
          <h3>Completed</h3>
          <p>{tasks.filter(t => t.status === 'done').length}</p>
        </div>
        <div className="stat-card">
          <h3>In Progress</h3>
          <p>{tasks.filter(t => t.status === 'inProgress').length}</p>
        </div>
      </div>
      {renderKanbanBoard()}
    </div>
  );
};

export default UserDashboard;
```

### Time Tracker Component

```javascript
// frontend/src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isActive, setIsActive] = useState(false);
  
  const API_URL = process.env.REACT_APP_API_URL;
  
  useEffect(() => {
    let interval = null;
    if (isActive) {
      interval = setInterval(() => {
        setSeconds(seconds => seconds + 1);
      }, 1000);
    } else if (!isActive && seconds !== 0) {
      clearInterval(interval);
    }
    return () => clearInterval(interval);
  }, [isActive, seconds]);
  
  const toggle = () => {
    setIsActive(!isActive);
  };
  
  const reset = async () => {
    if (seconds > 0) {
      try {
        const token = localStorage.getItem('token');
        await axios.post(
          `${API_URL}/api/tasks/${taskId}/time`,
          { duration: seconds },
          { headers: { Authorization: `Bearer ${token}` } }
        );
      } catch (error) {
        console.error('Error saving time:', error);
      }
    }
    setSeconds(0);
    setIsActive(false);
  };
  
  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const remainingSeconds = secs % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
  };
  
  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <div className="timer-controls">
        <button onClick={toggle}>
          {isActive ? 'Pause' : 'Start'}
        </button>
        <button onClick={reset}>Save & Reset</button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [alerts, setAlerts] = useState([]);
  
  const API_URL = process.env.REACT_APP_API_URL;
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;
  
  useEffect(() => {
    fetchAnalytics();
    fetchAlerts();
  }, []);
  
  const fetchAnalytics = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(`${API_URL}/api/admin/analytics`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      setAnalytics(response.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };
  
  const fetchAlerts = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(`${API_URL}/api/admin/alerts`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      setAlerts(response.data.alerts);
    } catch (error) {
      console.error('Error fetching alerts:', error);
    }
  };
  
  const checkBurnout = async (userId) => {
    try {
      const token = localStorage.getItem('token');
      const userStats = await axios.get(`${API_URL}/api/users/${userId}/workload`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      const burnoutResponse = await axios.post(
        `${ML_API_URL}/predict-burnout`,
        userStats.data
      );
      
      return burnoutResponse.data;
    } catch (error) {
      console.error('Error checking burnout:', error);
    }
  };
  
  return (
    <div className="admin-dashboard">
      <h1>Admin Analytics</h1>
      
      {analytics && (
        <div className="analytics-grid">
          <div className="metric-card">
            <h3>Total Users</h3>
            <p className="metric-value">{analytics.totalUsers}</p>
          </div>
          <div className="metric-card">
            <h3>Active Tasks</h3>
            <p className="metric-value">{analytics.activeTasks}</p>
          </div>
          <div className="metric-card">
            <h3>Open Tickets</h3>
            <p className="metric-value">{analytics.openTickets}</p>
          </div>
          <div className="metric-card">
            <h3>Completion Rate</h3>
            <p className="metric-value">{analytics.completionRate}%</p>
          </div>
        </div>
      )}
      
      <div className="alerts-section">
        <h2>Security Alerts</h2>
        {alerts.map(alert => (
          <div key={alert._id} className={`alert alert-${alert.severity}`}>
            <h4>{alert.title}</h4>
            <p>{alert.message}</p>
            <span className="alert-time">{new Date(alert.createdAt).toLocaleString()}</span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Database Models (MongoDB/Mongoose)

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  lastLogin: Date,
  loginAttempts: {
    type: Number,
    default: 0
  },
  isActive: {
    type: Boolean,
    default: true
  }
}, { timestamps: true });

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  status: {
    type: String,
    enum: ['todo', 'inProgress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
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
  dueDate: Date,
  completedAt: Date,
  timeSpent: {
    type: Number,
    default: 0
  }
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'urgent']
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'inProgress', 'resolved', 'closed'],
    default: 'open'
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  suggestedAssignee: String,
  resolution: String,
  resolvedAt: Date
}, { timestamps: true });

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### API Request Pattern with Auth

```javascript
// frontend/src/utils/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const api = axios.create({
  baseURL: API_URL
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle auth errors
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;

// Usage
import api from './utils/api';

const fetchData = async () => {
  const response = await api.get('/api/tasks/my-tasks');
  return response.data;
};
```

### AI Integration Pattern

```javascript
// backend/services/aiService.js
const axios = require('axios');

class AIService {
  constructor() {
    this.mlServiceUrl = process.env.ML_SERVICE_URL;
  }
  
  async classifyTicket(title, description) {
    try {
      const response = await axios.post(
        `${this.mlServiceUrl}/classify-ticket`,
        { title, description }
      );
      return response.data;
    } catch (error) {
      console.error('ML service error:', error);
      return { category: 'general', confidence: 0 };
    }
  }
  
  async detectAnomaly(userBehavior) {
    try {
      const response = await axios.post(
        `${this.mlServiceUrl}/detect-anomaly`,
        userBehavior
      );
