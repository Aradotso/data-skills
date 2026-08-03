---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task management, ticket routing, risk detection, and burnout prediction
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement AI-powered task and user management system"
  - "create user management dashboard with anomaly detection"
  - "build ticket management with AI classification"
  - "add burnout detection and risk prediction to user system"
  - "integrate AI analytics for enterprise user tracking"
  - "deploy full-stack user management with ML insights"
  - "configure JWT authentication for user management app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user/task management with machine learning capabilities. The system provides role-based access control, Kanban task tracking, support ticket management, and AI-powered insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

**Architecture:**
- **Frontend**: React.js with JWT authentication
- **Backend**: Node.js with REST APIs
- **ML Service**: FastAPI with scikit-learn and River (online learning)
- **Database**: MongoDB
- **AI Features**: Ticket classification, risk prediction, anomaly detection, burnout detection

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB instance (local or cloud)
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
npm start  # Runs on http://localhost:5000

# ML Service setup (in new terminal)
cd ml-service
pip install -r requirements.txt
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup (in new terminal)
cd frontend
npm install
npm start  # Runs on http://localhost:3000
```

## Configuration

### Backend Environment Variables

Create `backend/.env`:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your-secret-key-here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

### ML Service Environment Variables

Create `ml-service/.env`:

```env
MODEL_PATH=./models
BACKEND_URL=http://localhost:5000
LOG_LEVEL=INFO
```

### Frontend Environment Variables

Create `frontend/.env`:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

## Backend API Examples

### User Authentication

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
    const { name, email, password, role } = req.body;
    
    // Check if user exists
    let user = await User.findOne({ email });
    if (user) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    // Create user
    user = new User({
      name,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate JWT
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ token, user: { id: user._id, name, email, role } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ token, user: { id: user._id, name: user.name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const auth = require('../middleware/auth');
const axios = require('axios');

const router = express.Router();

// Create task
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.id,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });
    
    await task.save();
    
    // Call ML service for risk prediction
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/predict-risk`, {
        taskId: task._id,
        priority: task.priority,
        dueDate: task.dueDate,
        userId: assignedTo
      });
      
      task.riskScore = mlResponse.data.riskScore;
      await task.save();
    } catch (mlError) {
      console.error('ML prediction failed:', mlError.message);
    }
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user tasks
router.get('/my-tasks', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
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
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Ticket Management with AI Classification

```javascript
// backend/routes/tickets.js
const express = require('express');
const Ticket = require('../models/Ticket');
const auth = require('../middleware/auth');
const axios = require('axios');

const router = express.Router();

// Create ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    const ticket = new Ticket({
      title,
      description,
      priority: priority || 'medium',
      createdBy: req.user.id,
      status: 'open'
    });
    
    // AI-based classification and routing
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        title,
        description,
        priority
      });
      
      ticket.category = mlResponse.data.category;
      ticket.assignedTo = mlResponse.data.suggestedAssignee;
      ticket.urgencyScore = mlResponse.data.urgencyScore;
    } catch (mlError) {
      console.error('AI classification failed:', mlError.message);
      ticket.category = 'general';
    }
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get all tickets (admin)
router.get('/', auth, async (req, res) => {
  try {
    if (req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Access denied' });
    }
    
    const tickets = await Ticket.find()
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

### AI Models Implementation

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import joblib
import numpy as np
from datetime import datetime
from river import anomaly, tree

app = FastAPI()

# Load or initialize models
try:
    risk_model = joblib.load('./models/risk_model.pkl')
except:
    # Initialize with default classifier
    from sklearn.ensemble import RandomForestClassifier
    risk_model = RandomForestClassifier(n_estimators=100)

# Online learning model for anomaly detection
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)

class TicketData(BaseModel):
    title: str
    description: str
    priority: str

class RiskPredictionData(BaseModel):
    taskId: str
    priority: str
    dueDate: str
    userId: str

class BurnoutData(BaseModel):
    userId: str
    tasksCompleted: int
    tasksInProgress: int
    avgTaskDuration: float
    workingHours: float

@app.post("/classify-ticket")
async def classify_ticket(data: TicketData):
    """AI-based ticket classification and routing"""
    try:
        # Simple keyword-based classification (can be replaced with ML model)
        keywords = {
            'technical': ['bug', 'error', 'crash', 'api', 'server', 'database'],
            'billing': ['payment', 'invoice', 'charge', 'subscription', 'refund'],
            'account': ['login', 'password', 'access', 'permission', 'profile'],
            'feature': ['feature', 'enhancement', 'request', 'improvement']
        }
        
        text = (data.title + ' ' + data.description).lower()
        
        category = 'general'
        max_score = 0
        
        for cat, words in keywords.items():
            score = sum(1 for word in words if word in text)
            if score > max_score:
                max_score = score
                category = cat
        
        # Calculate urgency score
        urgency_map = {'low': 1, 'medium': 2, 'high': 3, 'critical': 4}
        urgency_score = urgency_map.get(data.priority.lower(), 2) * (max_score + 1)
        
        return {
            'category': category,
            'suggestedAssignee': None,  # Can be enhanced with user-skill matching
            'urgencyScore': urgency_score
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(data: RiskPredictionData):
    """Predict task completion risk"""
    try:
        # Calculate days until due date
        due_date = datetime.fromisoformat(data.dueDate.replace('Z', '+00:00'))
        days_remaining = (due_date - datetime.now()).days
        
        # Simple risk calculation (can be replaced with trained model)
        priority_weight = {'low': 0.2, 'medium': 0.5, 'high': 0.8, 'critical': 1.0}
        
        base_risk = priority_weight.get(data.priority.lower(), 0.5)
        
        if days_remaining < 0:
            risk_score = 1.0  # Overdue
        elif days_remaining < 2:
            risk_score = base_risk * 0.9
        elif days_remaining < 7:
            risk_score = base_risk * 0.7
        else:
            risk_score = base_risk * 0.4
        
        return {'riskScore': min(risk_score, 1.0)}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(user_behavior: dict):
    """Detect anomalous user behavior"""
    try:
        # Extract features
        features = {
            'login_hour': user_behavior.get('loginHour', 9),
            'actions_per_minute': user_behavior.get('actionsPerMinute', 2),
            'failed_logins': user_behavior.get('failedLogins', 0),
            'data_access_count': user_behavior.get('dataAccessCount', 10)
        }
        
        # Online anomaly detection
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.7
        
        return {
            'isAnomaly': is_anomaly,
            'anomalyScore': float(score),
            'message': 'Suspicious activity detected' if is_anomaly else 'Normal behavior'
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(data: BurnoutData):
    """Predict user burnout risk"""
    try:
        # Calculate burnout indicators
        task_load = data.tasksInProgress / max(data.tasksCompleted, 1)
        work_intensity = data.workingHours / 40  # Assuming 40hr work week
        
        burnout_score = 0.0
        
        # High workload indicator
        if task_load > 2:
            burnout_score += 0.3
        elif task_load > 1.5:
            burnout_score += 0.2
        
        # Long working hours
        if work_intensity > 1.5:
            burnout_score += 0.4
        elif work_intensity > 1.2:
            burnout_score += 0.2
        
        # Slow task completion
        if data.avgTaskDuration > 5:  # days
            burnout_score += 0.3
        
        burnout_score = min(burnout_score, 1.0)
        
        risk_level = 'high' if burnout_score > 0.7 else 'medium' if burnout_score > 0.4 else 'low'
        
        return {
            'burnoutScore': burnout_score,
            'riskLevel': risk_level,
            'recommendation': 'Consider redistributing tasks' if burnout_score > 0.6 else 'Monitor workload'
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Frontend React Components

### User Dashboard with Kanban Board

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks/my-tasks`,
        { headers: { Authorization: `Bearer ${token}` } }
      );

      // Organize tasks by status
      const organized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'inProgress'),
        done: response.data.filter(t => t.status === 'done')
      };

      setTasks(organized);
      setLoading(false);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
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
      
      fetchTasks(); // Refresh tasks
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" style={{
      border: '1px solid #ddd',
      borderRadius: '8px',
      padding: '12px',
      marginBottom: '12px',
      backgroundColor: task.riskScore > 0.7 ? '#fff3cd' : 'white'
    }}>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div>
        <span className={`priority-badge priority-${task.priority}`}>
          {task.priority}
        </span>
        {task.riskScore && (
          <span className="risk-badge">
            Risk: {(task.riskScore * 100).toFixed(0)}%
          </span>
        )}
      </div>
      <div className="task-actions">
        {task.status === 'todo' && (
          <button onClick={() => updateTaskStatus(task._id, 'inProgress')}>
            Start
          </button>
        )}
        {task.status === 'inProgress' && (
          <>
            <button onClick={() => updateTaskStatus(task._id, 'todo')}>
              Back to Todo
            </button>
            <button onClick={() => updateTaskStatus(task._id, 'done')}>
              Complete
            </button>
          </>
        )}
      </div>
    </div>
  );

  if (loading) return <div>Loading...</div>;

  return (
    <div className="kanban-board" style={{ display: 'flex', gap: '20px' }}>
      <div className="kanban-column" style={{ flex: 1 }}>
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      
      <div className="kanban-column" style={{ flex: 1 }}>
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      
      <div className="kanban-column" style={{ flex: 1 }}>
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeUsers: 0,
    totalTasks: 0,
    completionRate: 0,
    highRiskTasks: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const token = localStorage.getItem('token');
      const headers = { Authorization: `Bearer ${token}` };

      const [usersRes, tasksRes, ticketsRes] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/admin/users`, { headers }),
        axios.get(`${process.env.REACT_APP_API_URL}/admin/tasks`, { headers }),
        axios.get(`${process.env.REACT_APP_API_URL}/tickets`, { headers })
      ]);

      const users = usersRes.data;
      const tasks = tasksRes.data;

      const completedTasks = tasks.filter(t => t.status === 'done').length;
      const highRisk = tasks.filter(t => t.riskScore > 0.7);

      setAnalytics({
        totalUsers: users.length,
        activeUsers: users.filter(u => u.isActive).length,
        totalTasks: tasks.length,
        completionRate: (completedTasks / tasks.length * 100).toFixed(1),
        highRiskTasks: highRisk
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h2>Analytics Dashboard</h2>
      
      <div className="metrics-grid" style={{ display: 'grid', gridTemplateColumns: 'repeat(4, 1fr)', gap: '20px' }}>
        <div className="metric-card">
          <h3>{analytics.totalUsers}</h3>
          <p>Total Users</p>
        </div>
        <div className="metric-card">
          <h3>{analytics.activeUsers}</h3>
          <p>Active Users</p>
        </div>
        <div className="metric-card">
          <h3>{analytics.totalTasks}</h3>
          <p>Total Tasks</p>
        </div>
        <div className="metric-card">
          <h3>{analytics.completionRate}%</h3>
          <p>Completion Rate</p>
        </div>
      </div>

      <div className="high-risk-section" style={{ marginTop: '30px' }}>
        <h3>High Risk Tasks ({analytics.highRiskTasks.length})</h3>
        <table style={{ width: '100%', borderCollapse: 'collapse' }}>
          <thead>
            <tr>
              <th>Task</th>
              <th>Assigned To</th>
              <th>Risk Score</th>
              <th>Due Date</th>
            </tr>
          </thead>
          <tbody>
            {analytics.highRiskTasks.map(task => (
              <tr key={task._id}>
                <td>{task.title}</td>
                <td>{task.assignedTo?.name}</td>
                <td>{(task.riskScore * 100).toFixed(0)}%</td>
                <td>{new Date(task.dueDate).toLocaleDateString()}</td>
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

## Common Patterns

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

module.exports = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};
```

### Role-Based Access Control

```javascript
// backend/middleware/checkRole.js
module.exports = (roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: 'Access denied' });
    }
    next();
  };
};

// Usage in routes
const checkRole = require('../middleware/checkRole');
router.get('/admin/users', auth, checkRole(['admin']), async (req, res) => {
  // Admin-only logic
});
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Configuration

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Connection Timeout

```javascript
// backend/utils/mlClient.js
const axios = require('axios');

const mlClient = axios.create({
  baseURL: process.env.ML_SERVICE_URL,
  timeout: 5000,
  headers: { 'Content-Type': 'application/json' }
});

// Add error handling
mlClient.interceptors.response.use(
  response => response,
  error => {
    console.error('ML service error:', error.message);
    return Promise.reject(error);
  }
);

module.exports = mlClient;
```

### Token Expiration Handling (Frontend)

```javascript
// frontend/src/utils/axios.js
import axios from 'axios';

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

export default api;
```
