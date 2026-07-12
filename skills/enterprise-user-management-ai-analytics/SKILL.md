---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and risk detection
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement task tracking with AI insights"
  - "create admin dashboard for user management"
  - "add AI-based ticket classification"
  - "build user management with burnout detection"
  - "configure JWT authentication for enterprise app"
  - "implement Kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System is a full-stack JavaScript application that combines traditional user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, user CRUD operations, authentication via JWT
- **Task Management**: Kanban boards, time tracking, assignment workflows
- **Support Tickets**: AI-powered classification, routing, and priority detection
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay forecasting
- **Admin Dashboard**: Organization-wide analytics, audit logs, security alerts

The system uses React for frontend, Node.js/Express for backend, MongoDB for data persistence, and FastAPI with scikit-learn/River for ML services.

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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
# Server runs at http://localhost:5000
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

# Start ML service
uvicorn main:app --reload --port 8000
# ML API runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
EOF

# Start frontend
npm start
# App runs at http://localhost:3000
```

## Key Architecture

### Backend API Structure

The Node.js backend follows REST conventions:

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const jwt = require('jsonwebtoken');
const cors = require('cors');
require('dotenv').config();

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Database connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
})
.then(() => console.log('MongoDB connected'))
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

module.exports = function(req, res, next) {
  const token = req.header('x-auth-token');
  
  if (!token) {
    return res.status(401).json({ msg: 'No token, authorization denied' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded.user;
    next();
  } catch (err) {
    res.status(401).json({ msg: 'Token is not valid' });
  }
};
```

### Role-Based Access Control

```javascript
// backend/middleware/admin.js
module.exports = function(req, res, next) {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ msg: 'Access denied: Admin only' });
  }
  next();
};

// Usage in routes
const auth = require('../middleware/auth');
const admin = require('../middleware/admin');

router.delete('/users/:id', [auth, admin], async (req, res) => {
  // Only authenticated admins can delete users
});
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const UserSchema = new mongoose.Schema({
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
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('User', UserSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
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
  dueDate: Date,
  timeSpent: {
    type: Number,
    default: 0 // in minutes
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', TaskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const TicketSchema = new mongoose.Schema({
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
    enum: ['technical', 'hr', 'facilities', 'other']
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical']
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  raisedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  aiClassification: {
    category: String,
    confidence: Number
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Ticket', TicketSchema);
```

## Backend API Routes

### Authentication Routes

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');
const auth = require('../middleware/auth');

// @route   POST /api/auth/register
// @desc    Register new user
// @access  Public
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, department } = req.body;
    
    let user = await User.findOne({ email });
    if (user) {
      return res.status(400).json({ msg: 'User already exists' });
    }
    
    user = new User({
      name,
      email,
      password,
      department
    });
    
    const salt = await bcrypt.genSalt(10);
    user.password = await bcrypt.hash(password, salt);
    
    await user.save();
    
    const payload = {
      user: {
        id: user.id,
        role: user.role
      }
    };
    
    jwt.sign(
      payload,
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE },
      (err, token) => {
        if (err) throw err;
        res.json({ token });
      }
    );
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// @route   POST /api/auth/login
// @desc    Authenticate user
// @access  Public
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    let user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ msg: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ msg: 'Invalid credentials' });
    }
    
    const payload = {
      user: {
        id: user.id,
        role: user.role
      }
    };
    
    jwt.sign(
      payload,
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE },
      (err, token) => {
        if (err) throw err;
        res.json({ token, user: { id: user.id, name: user.name, role: user.role } });
      }
    );
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// @route   GET /api/auth/user
// @desc    Get current user
// @access  Private
router.get('/user', auth, async (req, res) => {
  try {
    const user = await User.findById(req.user.id).select('-password');
    res.json(user);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

module.exports = router;
```

### Task Management Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const auth = require('../middleware/auth');
const admin = require('../middleware/admin');

// @route   GET /api/tasks
// @desc    Get all tasks for user
// @access  Private
router.get('/', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// @route   POST /api/tasks
// @desc    Create new task
// @access  Private (Admin)
router.post('/', [auth, admin], async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const newTask = new Task({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      createdBy: req.user.id
    });
    
    const task = await newTask.save();
    res.json(task);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// @route   PUT /api/tasks/:id
// @desc    Update task status
// @access  Private
router.put('/:id', auth, async (req, res) => {
  try {
    const { status, timeSpent } = req.body;
    
    let task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ msg: 'Task not found' });
    }
    
    // Users can only update their own tasks
    if (task.assignedTo.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(401).json({ msg: 'Not authorized' });
    }
    
    if (status) task.status = status;
    if (timeSpent) task.timeSpent = timeSpent;
    
    await task.save();
    res.json(task);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

module.exports = router;
```

### Ticket Routes with AI Integration

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const auth = require('../middleware/auth');

// @route   POST /api/tickets
// @desc    Create new ticket with AI classification
// @access  Private
router.post('/', auth, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Call ML service for AI classification
    let aiClassification = null;
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { title, description }
      );
      aiClassification = mlResponse.data;
    } catch (mlErr) {
      console.error('ML service error:', mlErr.message);
    }
    
    const newTicket = new Ticket({
      title,
      description,
      raisedBy: req.user.id,
      category: aiClassification?.category,
      priority: aiClassification?.priority,
      aiClassification
    });
    
    const ticket = await newTicket.save();
    res.json(ticket);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// @route   GET /api/tickets
// @desc    Get user's tickets
// @access  Private
router.get('/', auth, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { raisedBy: req.user.id };
    
    const tickets = await Ticket.find(query)
      .populate('raisedBy', 'name email')
      .populate('assignedTo', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

module.exports = router;
```

## ML Service (FastAPI)

### Main ML Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from typing import Optional, List
import os

app = FastAPI(title="Enterprise AI Analytics Service")

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models (initialize on startup)
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_logins: int
    off_hours_access: int
    data_download_volume: float
    permission_changes: int

class BurnoutRequest(BaseModel):
    user_id: str
    tasks_count: int
    avg_task_time: float
    overtime_hours: float
    pending_tasks: int

@app.get("/")
def root():
    return {"service": "Enterprise AI Analytics", "status": "running"}

@app.post("/classify-ticket")
async def classify_ticket(request: TicketRequest):
    """
    Classify support ticket using NLP
    """
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple rule-based classification (replace with trained model)
        category = "other"
        priority = "medium"
        confidence = 0.7
        
        # Category classification
        if any(word in text for word in ['password', 'login', 'access', 'network', 'software']):
            category = "technical"
            confidence = 0.85
        elif any(word in text for word in ['leave', 'salary', 'hr', 'policy']):
            category = "hr"
            confidence = 0.80
        elif any(word in text for word in ['room', 'desk', 'facilities', 'maintenance']):
            category = "facilities"
            confidence = 0.75
        
        # Priority detection
        if any(word in text for word in ['urgent', 'critical', 'emergency', 'asap']):
            priority = "high"
        elif any(word in text for word in ['important', 'soon']):
            priority = "medium"
        else:
            priority = "low"
        
        return {
            "category": category,
            "priority": priority,
            "confidence": confidence
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """
    Predict user risk score based on behavioral patterns
    """
    try:
        # Calculate risk score (0-100)
        risk_score = 0
        
        # Failed login attempts
        if request.failed_logins > 5:
            risk_score += 30
        elif request.failed_logins > 2:
            risk_score += 15
        
        # Off-hours access
        if request.off_hours_access > 10:
            risk_score += 25
        elif request.off_hours_access > 5:
            risk_score += 10
        
        # Data download volume (in MB)
        if request.data_download_volume > 1000:
            risk_score += 20
        elif request.data_download_volume > 500:
            risk_score += 10
        
        # Permission changes
        if request.permission_changes > 3:
            risk_score += 25
        
        risk_level = "low"
        if risk_score > 70:
            risk_level = "critical"
        elif risk_score > 50:
            risk_level = "high"
        elif risk_score > 30:
            risk_level = "medium"
        
        return {
            "user_id": request.user_id,
            "risk_score": min(risk_score, 100),
            "risk_level": risk_level,
            "recommendations": get_risk_recommendations(risk_score)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutRequest):
    """
    Detect employee burnout risk
    """
    try:
        burnout_score = 0
        
        # High task count
        if request.tasks_count > 20:
            burnout_score += 30
        elif request.tasks_count > 15:
            burnout_score += 15
        
        # Average task time (hours per task)
        if request.avg_task_time > 8:
            burnout_score += 25
        elif request.avg_task_time > 5:
            burnout_score += 10
        
        # Overtime hours per week
        if request.overtime_hours > 15:
            burnout_score += 30
        elif request.overtime_hours > 10:
            burnout_score += 15
        
        # Pending tasks
        if request.pending_tasks > 10:
            burnout_score += 15
        
        risk_level = "low"
        if burnout_score > 70:
            risk_level = "critical"
        elif burnout_score > 50:
            risk_level = "high"
        elif burnout_score > 30:
            risk_level = "medium"
        
        return {
            "user_id": request.user_id,
            "burnout_score": min(burnout_score, 100),
            "risk_level": risk_level,
            "recommendations": get_burnout_recommendations(burnout_score)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_risk_recommendations(score: int) -> List[str]:
    recommendations = []
    if score > 70:
        recommendations.append("Immediate security review required")
        recommendations.append("Consider account suspension pending investigation")
    elif score > 50:
        recommendations.append("Monitor user activity closely")
        recommendations.append("Review access permissions")
    elif score > 30:
        recommendations.append("Schedule routine security check")
    return recommendations

def get_burnout_recommendations(score: int) -> List[str]:
    recommendations = []
    if score > 70:
        recommendations.append("Urgent: Redistribute workload immediately")
        recommendations.append("Schedule mandatory time off")
    elif score > 50:
        recommendations.append("Review task assignments")
        recommendations.append("Consider additional resources")
    elif score > 30:
        recommendations.append("Monitor workload trends")
    return recommendations
```

### ML Service Requirements

```python
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
scikit-learn==1.3.2
river==0.19.0
numpy==1.24.3
joblib==1.3.2
python-dotenv==1.0.0
```

## Frontend Integration

### API Service Layer

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';
const ML_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

// Set auth token
export const setAuthToken = (token) => {
  if (token) {
    axios.defaults.headers.common['x-auth-token'] = token;
    localStorage.setItem('token', token);
  } else {
    delete axios.defaults.headers.common['x-auth-token'];
    localStorage.removeItem('token');
  }
};

// Authentication
export const login = async (email, password) => {
  const response = await axios.post(`${API_URL}/auth/login`, { email, password });
  if (response.data.token) {
    setAuthToken(response.data.token);
  }
  return response.data;
};

export const register = async (userData) => {
  const response = await axios.post(`${API_URL}/auth/register`, userData);
  if (response.data.token) {
    setAuthToken(response.data.token);
  }
  return response.data;
};

export const getCurrentUser = async () => {
  const response = await axios.get(`${API_URL}/auth/user`);
  return response.data;
};

// Tasks
export const getTasks = async () => {
  const response = await axios.get(`${API_URL}/tasks`);
  return response.data;
};

export const createTask = async (taskData) => {
  const response = await axios.post(`${API_URL}/tasks`, taskData);
  return response.data;
};

export const updateTask = async (taskId, updates) => {
  const response = await axios.put(`${API_URL}/tasks/${taskId}`, updates);
  return response.data;
};

// Tickets
export const getTickets = async () => {
  const response = await axios.get(`${API_URL}/tickets`);
  return response.data;
};

export const createTicket = async (ticketData) => {
  const response = await axios.post(`${API_URL}/tickets`, ticketData);
  return response.data;
};

// AI Analytics
export const getRiskAnalysis = async (userId, data) => {
  const response = await axios.post(`${ML_URL}/predict-risk`, {
    user_id: userId,
    ...data
  });
  return response.data;
};

export const getBurnoutAnalysis = async (userId, data) => {
  const response = await axios.post(`${ML_URL}/detect-burnout`, {
    user_id: userId,
    ...data
  });
  return response.data;
};
```

### React Component Example: Task Board

```javascript
// frontend/src/components/TaskBoard.js
import React, { useState, useEffect } from 'react';
import { getTasks, updateTask } from '../services/api';
import './TaskBoard.css';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const data = await getTasks();
      const grouped = {
        todo: data.filter(t => t.status === 'todo'),
        'in-progress': data.filter(t => t.status === 'in-progress'),
        done: data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await updateTask(taskId, { status: newStatus });
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="task-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div key={status} className="task-column">
          <h3>{status.toUpperCase()}</h3>
          <div className="task-list">
            {tasks[status].map(task => (
              <div key={task._id} className="task-card">
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <span className={`priority ${task.priority}`}>
                  {task.priority}
                </span>
                <div className="task-actions">
                  {status !== 'done' && (
                    <button
                      onClick={() => handleStatusChange(
                        task._id,
                        status === 'todo' ? 'in-progress' : 'done'
                      )}
                    >
                      {status === 'todo' ? 'Start' : 'Complete'}
                    </button>
                  )}
                </div>
              </div>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default TaskBoard;
```

### React Component: AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import { getRiskAnalysis, getBurnoutAnalysis } from '../services/api';
import './AIAnalytics.css';

const AIAnalytics = ({ userId }) => {
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Example data - replace with actual metrics from your database
      const riskMetrics = {
        failed_logins: 2,
        off_hours_access: 3,
        data_download_volume: 250,
        permission_changes: 1
      };
      
      const burnoutMetrics = {
        tasks_count: 12,
        avg_task_time: 4.5,
        overtime_hours: 8,
        pending_tasks: 5
      };

      const [risk, burnout] = await Promise.all([
        getRiskAnalysis(userId, riskMetrics),
        getBurnoutAnalysis(userId, burnoutMetrics)
      ]);

      setRiskData(risk);
      setBurnoutData(burnout);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Analytics</h2>
      
      {riskData && (
        <div className={`analytics-card risk-${riskData.risk_level}`}>
          <h3>Security Risk Assessment</h3>
          <div className="score">
            <span className="score-value">{riskData.risk_score}</span>
            <span className="score-label">Risk Score</span>
          </div>
          <p className="risk-level">Level: {riskData.risk_level.toUpperCase()}</p>
          <div className="recommendations">
            <h4>Recommendations:</h4>
            <ul>
              {riskData.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        </div>
      )}

      {burnoutData && (
        <div className={`analytics-card burnout-${burnoutData.risk_level}`}>
          <h3>Burnout Risk Assessment</h3>
          <div className="score">
            <span className="score-value">{burnoutData.burnout_score}</span>
            <span className="score-label">Burnout Score</span>
          </div>
          
