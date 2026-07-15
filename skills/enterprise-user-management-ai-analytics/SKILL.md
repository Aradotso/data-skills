---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task management, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create task management with burnout detection"
  - "build user management system with AI insights"
  - "integrate AI ticket classification system"
  - "add anomaly detection to user management"
  - "implement risk prediction for users"
  - "create Kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with AI-driven insights. It provides role-based access control, Kanban-style task tracking, support ticket management, and machine learning features including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

**Key Components:**
- **Frontend**: React.js dashboard for admins and users
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based AI/ML microservice with scikit-learn and River
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

```bash
# Required
node --version  # v14+
python --version  # 3.8+
mongod --version  # 4.4+
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
```

Create `.env` file:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRES_IN=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Runs on http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `ml-service/.env`:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# Runs on http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `frontend/.env`:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Runs on http://localhost:3000
```

## Architecture

### Backend API Structure

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
const User = require('../models/User');

module.exports = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user) {
      throw new Error();
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid authentication token' });
  }
};
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
  role: { 
    type: String, 
    enum: ['admin', 'user', 'manager'], 
    default: 'user' 
  },
  department: String,
  status: { 
    type: String, 
    enum: ['active', 'inactive', 'suspended'], 
    default: 'active' 
  },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

// Compare password method
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
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  dueDate: Date,
  timeTracked: { type: Number, default: 0 }, // in seconds
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
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
    enum: ['technical', 'billing', 'feature_request', 'bug', 'other'],
    default: 'other'
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'] 
  },
  status: { 
    type: String, 
    enum: ['open', 'in_progress', 'resolved', 'closed'], 
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
  aiClassification: {
    category: String,
    confidence: Number,
    suggestedAssignee: String
  },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## API Endpoints

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

    const token = jwt.sign(
      { userId: user._id, role: user.role }, 
      process.env.JWT_SECRET, 
      { expiresIn: process.env.JWT_EXPIRES_IN }
    );

    res.status(201).json({ 
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

    const token = jwt.sign(
      { userId: user._id, role: user.role }, 
      process.env.JWT_SECRET, 
      { expiresIn: process.env.JWT_EXPIRES_IN }
    );

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
const auth = require('../middleware/auth');
const Task = require('../models/Task');
const router = express.Router();

// Get all tasks (filtered by user or all for admin)
router.get('/', auth, async (req, res) => {
  try {
    const filter = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user._id };
    
    const tasks = await Task.find(filter)
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

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.status = status;
    task.updatedAt = new Date();
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Track time on task
router.patch('/:id/time', auth, async (req, res) => {
  try {
    const { seconds } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.timeTracked += seconds;
    task.updatedAt = new Date();
    await task.save();
    
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
const auth = require('../middleware/auth');
const Ticket = require('../models/Ticket');
const router = express.Router();

// Create ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Call ML service for classification
    let aiClassification = null;
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { title, description }
      );
      aiClassification = mlResponse.data;
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }

    const ticket = new Ticket({
      title,
      description,
      createdBy: req.user._id,
      category: aiClassification?.category || 'other',
      priority: aiClassification?.priority || 'medium',
      aiClassification
    });

    await ticket.save();
    await ticket.populate('createdBy', 'name email');
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Get all tickets
router.get('/', auth, async (req, res) => {
  try {
    const filter = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user._id };
    
    const tickets = await Ticket.find(filter)
      .populate('createdBy assignedTo', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI Main Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, Dict, List
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise ML Service")

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    priority: str
    confidence: float
    suggestedAssignee: Optional[str]

class RiskPredictionRequest(BaseModel):
    userId: str
    failedLogins: int
    suspiciousActivities: int
    lastLoginDaysAgo: int
    tasksOverdue: int

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    tasksAssigned: int
    tasksCompleted: int
    avgHoursPerDay: float
    overtimeHours: float
    missedDeadlines: int

@app.get("/")
def read_root():
    return {"message": "Enterprise ML Service", "status": "running"}

@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket using NLP"""
    try:
        # Simple keyword-based classification (replace with trained model)
        text = f"{request.title} {request.description}".lower()
        
        category = "other"
        priority = "medium"
        confidence = 0.75
        
        # Category classification
        if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = "bug"
            priority = "high"
        elif any(word in text for word in ['feature', 'enhancement', 'request']):
            category = "feature_request"
            priority = "low"
        elif any(word in text for word in ['billing', 'payment', 'invoice']):
            category = "billing"
            priority = "medium"
        elif any(word in text for word in ['technical', 'api', 'integration']):
            category = "technical"
            priority = "high"
        
        # Priority boost for urgent keywords
        if any(word in text for word in ['urgent', 'critical', 'asap', 'immediately']):
            priority = "critical"
            confidence = 0.9
        
        return TicketClassificationResponse(
            category=category,
            priority=priority,
            confidence=confidence,
            suggestedAssignee=None
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior patterns"""
    try:
        # Calculate risk score (0-100)
        risk_score = 0
        risk_factors = []
        
        # Failed login attempts
        if request.failedLogins > 5:
            risk_score += 30
            risk_factors.append("Multiple failed login attempts")
        elif request.failedLogins > 2:
            risk_score += 15
        
        # Suspicious activities
        if request.suspiciousActivities > 3:
            risk_score += 25
            risk_factors.append("Suspicious activity detected")
        
        # Inactive account
        if request.lastLoginDaysAgo > 30:
            risk_score += 20
            risk_factors.append("Account inactive for extended period")
        
        # Overdue tasks
        if request.tasksOverdue > 5:
            risk_score += 15
            risk_factors.append("Multiple overdue tasks")
        
        risk_level = "low"
        if risk_score >= 70:
            risk_level = "critical"
        elif risk_score >= 50:
            risk_level = "high"
        elif risk_score >= 30:
            risk_level = "medium"
        
        return {
            "userId": request.userId,
            "riskScore": min(risk_score, 100),
            "riskLevel": risk_level,
            "riskFactors": risk_factors,
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze employee burnout risk"""
    try:
        burnout_score = 0
        warnings = []
        
        # Workload analysis
        if request.tasksAssigned > 20:
            burnout_score += 25
            warnings.append("High task volume")
        
        # Completion rate
        completion_rate = request.tasksCompleted / request.tasksAssigned if request.tasksAssigned > 0 else 1
        if completion_rate < 0.6:
            burnout_score += 20
            warnings.append("Low task completion rate")
        
        # Work hours
        if request.avgHoursPerDay > 10:
            burnout_score += 30
            warnings.append("Excessive work hours")
        
        # Overtime
        if request.overtimeHours > 20:
            burnout_score += 20
            warnings.append("High overtime hours")
        
        # Missed deadlines
        if request.missedDeadlines > 3:
            burnout_score += 15
            warnings.append("Frequent missed deadlines")
        
        burnout_level = "low"
        if burnout_score >= 70:
            burnout_level = "critical"
        elif burnout_score >= 50:
            burnout_level = "high"
        elif burnout_score >= 30:
            burnout_level = "moderate"
        
        recommendations = []
        if burnout_level in ["high", "critical"]:
            recommendations.extend([
                "Reduce task assignment",
                "Schedule one-on-one check-in",
                "Encourage time off"
            ])
        
        return {
            "userId": request.userId,
            "burnoutScore": min(burnout_score, 100),
            "burnoutLevel": burnout_level,
            "warnings": warnings,
            "recommendations": recommendations,
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(data: Dict):
    """Detect anomalies in user behavior"""
    try:
        # Simple anomaly detection based on thresholds
        anomalies = []
        
        login_time = data.get('loginTime', '12:00')
        hour = int(login_time.split(':')[0])
        if hour < 6 or hour > 22:
            anomalies.append("Unusual login time")
        
        if data.get('loginLocation') and data.get('previousLocation'):
            if data['loginLocation'] != data['previousLocation']:
                anomalies.append("Login from new location")
        
        if data.get('dataAccessVolume', 0) > 1000:
            anomalies.append("Unusually high data access")
        
        is_anomalous = len(anomalies) > 0
        
        return {
            "isAnomalous": is_anomalous,
            "anomalies": anomalies,
            "confidence": 0.8 if is_anomalous else 0.95,
            "timestamp": datetime.utcnow().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Requirements for ML Service

```txt
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
numpy==1.24.3
scikit-learn==1.3.2
joblib==1.3.2
python-dotenv==1.0.0
river==0.20.0
```

## Frontend Integration

### API Service Layer

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add auth token to requests
api.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export const authService = {
  login: (email, password) => 
    api.post('/auth/login', { email, password }),
  
  register: (userData) => 
    api.post('/auth/register', userData)
};

export const taskService = {
  getTasks: () => api.get('/tasks'),
  
  createTask: (taskData) => api.post('/tasks', taskData),
  
  updateStatus: (taskId, status) => 
    api.patch(`/tasks/${taskId}/status`, { status }),
  
  trackTime: (taskId, seconds) => 
    api.patch(`/tasks/${taskId}/time`, { seconds })
};

export const ticketService = {
  getTickets: () => api.get('/tickets'),
  
  createTicket: (ticketData) => api.post('/tickets', ticketData),
  
  updateTicket: (ticketId, updates) => 
    api.patch(`/tickets/${ticketId}`, updates)
};

export const mlService = {
  getRiskPrediction: (userId, data) => 
    axios.post(`${ML_API_URL}/predict-risk`, { userId, ...data }),
  
  getBurnoutAnalysis: (userId, data) => 
    axios.post(`${ML_API_URL}/analyze-burnout`, { userId, ...data }),
  
  detectAnomaly: (data) => 
    axios.post(`${ML_API_URL}/detect-anomaly`, data)
};

export default api;
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskService } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskService.getTasks();
      const groupedTasks = {
        todo: response.data.filter(t => t.status === 'todo'),
        in_progress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(groupedTasks);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleDragStart = (e, task) => {
    e.dataTransfer.setData('taskId', task._id);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    
    try {
      await taskService.updateStatus(taskId, newStatus);
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const renderColumn = (status, title) => (
    <div 
      className="kanban-column"
      onDrop={(e) => handleDrop(e, status)}
      onDragOver={handleDragOver}
    >
      <h3>{title} ({tasks[status].length})</h3>
      <div className="task-list">
        {tasks[status].map(task => (
          <div
            key={task._id}
            className="task-card"
            draggable
            onDragStart={(e) => handleDragStart(e, task)}
          >
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <div className="task-meta">
              <span className={`priority ${task.priority}`}>
                {task.priority}
              </span>
              {task.timeTracked > 0 && (
                <span className="time-tracked">
                  {Math.floor(task.timeTracked / 3600)}h
                </span>
              )}
            </div>
          </div>
        ))}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do')}
      {renderColumn('in_progress', 'In Progress')}
      {renderColumn('done', 'Done')}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard Component

```javascript
// frontend/src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import { mlService } from '../services/api';
import './AIAnalyticsDashboard.css';

const AIAnalyticsDashboard = ({ userId }) => {
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch risk prediction
      const riskResponse = await mlService.getRiskPrediction(userId, {
        failedLogins: 1,
        suspiciousActivities: 0,
        lastLoginDaysAgo: 2,
        tasksOverdue: 3
      });
      setRiskData(riskResponse.data);

      // Fetch burnout analysis
      const burnoutResponse = await mlService.getBurnoutAnalysis(userId, {
        tasksAssigned: 15,
        tasksCompleted: 12,
        avgHoursPerDay: 8.5,
        overtimeHours: 10,
        missedDeadlines: 2
      });
      setBurnoutData(burnoutResponse.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI-Powered Insights</h2>
      
      <div className="analytics-grid">
        {/* Risk Analysis */}
        <div className="analytics-card">
          <h3>Risk Assessment</h3>
          <div className={`risk-score ${riskData?.riskLevel}`}>
            {riskData?.riskScore}/100
          </div>
          <p className="risk-level">Level: {riskData?.riskLevel}</p>
          {riskData?.riskFactors?.length > 0 && (
            <div className="risk-factors">
              <h4>Risk Factors:</h4>
              <ul>
                {riskData.riskFactors.map((factor, idx) => (
                  <li key={idx}>{factor}</li>
                ))}
              </ul>
            </div>
          )}
        </div>

        {/* Burnout Analysis */}
        <div className="analytics-card
