---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and predictive insights
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics for user behavior prediction
  - implement JWT authentication for user management
  - create a task management system with Kanban board
  - add AI-based ticket classification and routing
  - build anomaly detection for user activities
  - set up ML service for burnout prediction
  - create admin dashboard with AI insights
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack JavaScript application that combines traditional user/task management with AI-powered analytics. It provides role-based access control, task tracking with Kanban boards, support ticket management, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

The system consists of three main components:
- **Frontend**: React.js application for user interfaces
- **Backend**: Node.js REST API with MongoDB
- **ML Service**: FastAPI service with scikit-learn and River for real-time ML predictions

## Installation

### Prerequisites

```bash
# Ensure you have the following installed
node --version  # v14+ required
npm --version
python --version  # Python 3.8+ for ML service
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

# Start backend server
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file for ML service
cat > .env << EOF
MODEL_PATH=./models
DATABASE_URL=${MONGODB_URI}
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
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
```

## Architecture

### Backend API Structure

The Node.js backend follows this typical structure:

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

// MongoDB Connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
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

## Key Components

### Authentication with JWT

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

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// @route   POST /api/auth/register
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;

    let user = await User.findOne({ email });
    if (user) {
      return res.status(400).json({ msg: 'User already exists' });
    }

    user = new User({ name, email, password, role: role || 'user' });

    const salt = await bcrypt.genSalt(10);
    user.password = await bcrypt.hash(password, salt);

    await user.save();

    const payload = { user: { id: user.id, role: user.role } };
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

// @route   POST /api/auth/login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;

    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ msg: 'Invalid credentials' });
    }

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ msg: 'Invalid credentials' });
    }

    const payload = { user: { id: user.id, role: user.role } };
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

module.exports = router;
```

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
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  avatar: String,
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: Date,
  isActive: {
    type: Boolean,
    default: true
  },
  metadata: {
    loginCount: { type: Number, default: 0 },
    failedLoginAttempts: { type: Number, default: 0 }
  }
});

module.exports = mongoose.model('User', UserSchema);
```

### Task Management

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
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
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  deadline: Date,
  timeTracked: {
    type: Number,
    default: 0  // in minutes
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: Date
});

module.exports = mongoose.model('Task', TaskSchema);
```

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const auth = require('../middleware/auth');
const Task = require('../models/Task');

// @route   GET /api/tasks
router.get('/', auth, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user.id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .populate('assignedBy', 'name')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// @route   POST /api/tasks
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, deadline } = req.body;

    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.id,
      priority,
      deadline
    });

    await task.save();
    await task.populate('assignedTo', 'name email');
    
    res.json(task);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// @route   PATCH /api/tasks/:id/status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ msg: 'Task not found' });
    }

    task.status = status;
    task.updatedAt = Date.now();
    await task.save();

    res.json(task);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

module.exports = router;
```

### Support Ticket System

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
    enum: ['technical', 'billing', 'account', 'feature-request', 'bug', 'other'],
    default: 'other'
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
    suggestedAssignee: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User'
    }
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: Date,
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', TicketSchema);
```

## ML Service Integration

### FastAPI ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List, Dict
import joblib
import numpy as np
from datetime import datetime
import logging

app = FastAPI(title="Enterprise User Management ML Service")

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Load or initialize models
try:
    risk_model = joblib.load('models/risk_model.pkl')
    anomaly_model = joblib.load('models/anomaly_model.pkl')
    burnout_model = joblib.load('models/burnout_model.pkl')
except:
    logger.warning("Models not found, will use placeholder logic")
    risk_model = None
    anomaly_model = None
    burnout_model = None

class UserBehavior(BaseModel):
    user_id: str
    login_frequency: int
    failed_login_attempts: int
    tasks_completed: int
    tasks_overdue: int
    avg_task_completion_time: float
    work_hours_per_day: float
    weekend_activity: bool
    late_night_activity: bool

class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    user_history: Optional[Dict] = None

class RiskPredictionResponse(BaseModel):
    user_id: str
    risk_score: float
    risk_level: str
    factors: List[str]
    recommendations: List[str]

@app.get("/")
def root():
    return {"status": "ML Service Running", "version": "1.0.0"}

@app.post("/predict/risk")
def predict_risk(behavior: UserBehavior) -> RiskPredictionResponse:
    """Predict user risk score based on behavior patterns"""
    try:
        # Calculate risk score
        risk_score = 0.0
        factors = []
        
        # Failed login attempts
        if behavior.failed_login_attempts > 3:
            risk_score += 30
            factors.append("High failed login attempts")
        
        # Task completion issues
        if behavior.tasks_overdue > behavior.tasks_completed * 0.3:
            risk_score += 25
            factors.append("High overdue task ratio")
        
        # Unusual work patterns
        if behavior.work_hours_per_day > 12:
            risk_score += 20
            factors.append("Excessive work hours")
        
        if behavior.late_night_activity:
            risk_score += 15
            factors.append("Frequent late night activity")
        
        # Determine risk level
        if risk_score >= 70:
            risk_level = "critical"
        elif risk_score >= 50:
            risk_level = "high"
        elif risk_score >= 30:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        # Generate recommendations
        recommendations = []
        if "High failed login attempts" in factors:
            recommendations.append("Review account security and reset password")
        if "High overdue task ratio" in factors:
            recommendations.append("Reassign tasks or provide additional support")
        if "Excessive work hours" in factors:
            recommendations.append("Monitor for burnout and adjust workload")
        
        return RiskPredictionResponse(
            user_id=behavior.user_id,
            risk_score=min(risk_score, 100),
            risk_level=risk_level,
            factors=factors,
            recommendations=recommendations
        )
    except Exception as e:
        logger.error(f"Risk prediction error: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict/anomaly")
def detect_anomaly(behavior: UserBehavior):
    """Detect anomalous user behavior"""
    try:
        anomalies = []
        
        # Check for unusual login patterns
        if behavior.failed_login_attempts > 5:
            anomalies.append({
                "type": "security",
                "description": "Unusual failed login attempts",
                "severity": "high"
            })
        
        # Check for productivity anomalies
        if behavior.tasks_completed == 0 and behavior.login_frequency > 10:
            anomalies.append({
                "type": "productivity",
                "description": "High login frequency but no task completion",
                "severity": "medium"
            })
        
        # Check for work-life balance issues
        if behavior.weekend_activity and behavior.work_hours_per_day > 10:
            anomalies.append({
                "type": "burnout_risk",
                "description": "Working excessive hours including weekends",
                "severity": "high"
            })
        
        return {
            "user_id": behavior.user_id,
            "is_anomalous": len(anomalies) > 0,
            "anomalies": anomalies,
            "timestamp": datetime.now().isoformat()
        }
    except Exception as e:
        logger.error(f"Anomaly detection error: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict/burnout")
def predict_burnout(behavior: UserBehavior):
    """Predict employee burnout risk"""
    try:
        burnout_score = 0.0
        indicators = []
        
        # Work hours
        if behavior.work_hours_per_day > 10:
            burnout_score += 35
            indicators.append("Excessive daily work hours")
        
        # Weekend work
        if behavior.weekend_activity:
            burnout_score += 25
            indicators.append("Regular weekend work")
        
        # Late night activity
        if behavior.late_night_activity:
            burnout_score += 20
            indicators.append("Frequent late night activity")
        
        # Task overload
        overdue_ratio = behavior.tasks_overdue / max(behavior.tasks_completed, 1)
        if overdue_ratio > 0.4:
            burnout_score += 20
            indicators.append("High task overdue ratio")
        
        # Determine burnout level
        if burnout_score >= 70:
            burnout_level = "critical"
            action = "Immediate intervention required"
        elif burnout_score >= 50:
            burnout_level = "high"
            action = "Schedule wellness check and reduce workload"
        elif burnout_score >= 30:
            burnout_level = "moderate"
            action = "Monitor closely and provide support"
        else:
            burnout_level = "low"
            action = "Continue regular monitoring"
        
        return {
            "user_id": behavior.user_id,
            "burnout_score": burnout_score,
            "burnout_level": burnout_level,
            "indicators": indicators,
            "recommended_action": action,
            "timestamp": datetime.now().isoformat()
        }
    except Exception as e:
        logger.error(f"Burnout prediction error: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/classify/ticket")
def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket and route to appropriate team"""
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple keyword-based classification
        if any(word in text for word in ['bug', 'error', 'crash', 'not working']):
            category = 'bug'
            priority = 'high'
        elif any(word in text for word in ['payment', 'billing', 'invoice', 'charge']):
            category = 'billing'
            priority = 'high'
        elif any(word in text for word in ['password', 'login', 'access', 'account']):
            category = 'account'
            priority = 'medium'
        elif any(word in text for word in ['feature', 'enhancement', 'suggestion']):
            category = 'feature-request'
            priority = 'low'
        else:
            category = 'technical'
            priority = 'medium'
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85,
            "suggested_team": category,
            "auto_response": f"Thank you for reporting this {category} issue. Our team will review it shortly."
        }
    except Exception as e:
        logger.error(f"Ticket classification error: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))
```

### Backend Integration with ML Service

```javascript
// backend/services/mlService.js
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL || 'http://localhost:8000';

class MLService {
  async predictRisk(userBehavior) {
    try {
      const response = await axios.post(`${ML_SERVICE_URL}/predict/risk`, userBehavior);
      return response.data;
    } catch (error) {
      console.error('ML Service Risk Prediction Error:', error.message);
      throw error;
    }
  }

  async detectAnomaly(userBehavior) {
    try {
      const response = await axios.post(`${ML_SERVICE_URL}/predict/anomaly`, userBehavior);
      return response.data;
    } catch (error) {
      console.error('ML Service Anomaly Detection Error:', error.message);
      throw error;
    }
  }

  async predictBurnout(userBehavior) {
    try {
      const response = await axios.post(`${ML_SERVICE_URL}/predict/burnout`, userBehavior);
      return response.data;
    } catch (error) {
      console.error('ML Service Burnout Prediction Error:', error.message);
      throw error;
    }
  }

  async classifyTicket(ticketData) {
    try {
      const response = await axios.post(`${ML_SERVICE_URL}/classify/ticket`, ticketData);
      return response.data;
    } catch (error) {
      console.error('ML Service Ticket Classification Error:', error.message);
      throw error;
    }
  }
}

module.exports = new MLService();
```

```javascript
// backend/routes/analytics.js
const express = require('express');
const router = express.Router();
const auth = require('../middleware/auth');
const adminAuth = require('../middleware/adminAuth');
const mlService = require('../services/mlService');
const User = require('../models/User');
const Task = require('../models/Task');

// @route   GET /api/analytics/user/:userId/risk
router.get('/user/:userId/risk', [auth, adminAuth], async (req, res) => {
  try {
    const user = await User.findById(req.params.userId);
    if (!user) {
      return res.status(404).json({ msg: 'User not found' });
    }

    const tasks = await Task.find({ assignedTo: user._id });
    
    const completedTasks = tasks.filter(t => t.status === 'done').length;
    const overdueTasks = tasks.filter(t => 
      t.deadline && new Date(t.deadline) < new Date() && t.status !== 'done'
    ).length;
    
    const avgCompletionTime = tasks
      .filter(t => t.timeTracked > 0)
      .reduce((sum, t) => sum + t.timeTracked, 0) / Math.max(tasks.length, 1);

    const userBehavior = {
      user_id: user._id.toString(),
      login_frequency: user.metadata.loginCount || 0,
      failed_login_attempts: user.metadata.failedLoginAttempts || 0,
      tasks_completed: completedTasks,
      tasks_overdue: overdueTasks,
      avg_task_completion_time: avgCompletionTime,
      work_hours_per_day: 8, // Calculate from actual time tracking
      weekend_activity: false, // Calculate from login logs
      late_night_activity: false // Calculate from login logs
    };

    const riskPrediction = await mlService.predictRisk(userBehavior);
    res.json(riskPrediction);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// @route   GET /api/analytics/user/:userId/burnout
router.get('/user/:userId/burnout', [auth, adminAuth], async (req, res) => {
  try {
    const user = await User.findById(req.params.userId);
    if (!user) {
      return res.status(404).json({ msg: 'User not found' });
    }

    const tasks = await Task.find({ assignedTo: user._id });
    
    const userBehavior = {
      user_id: user._id.toString(),
      login_frequency: user.metadata.loginCount || 0,
      failed_login_attempts: user.metadata.failedLoginAttempts || 0,
      tasks_completed: tasks.filter(t => t.status === 'done').length,
      tasks_overdue: tasks.filter(t => 
        t.deadline && new Date(t.deadline) < new Date() && t.status !== 'done'
      ).length,
      avg_task_completion_time: 60,
      work_hours_per_day: 9,
      weekend_activity: true,
      late_night_activity: true
    };

    const burnoutPrediction = await mlService.predictBurnout(userBehavior);
    res.json(burnoutPrediction);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

module.exports = router;
```

## Frontend Integration

### React Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['x-auth-token'] = token;
      loadUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const loadUser = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(res.data);
    } catch (err) {
      console.error('Load user error:', err);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    try {
      const res = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
        email,
        password
      });
      
      setToken(res.data.token);
      setUser(res.data.user);
      localStorage.setItem('token', res.data.token);
      axios.defaults.headers.common['x-auth-token'] = res.data.token;
      
      return { success: true };
    } catch (err) {
      return { 
        success: false, 
        error: err.response?.data?.msg || 'Login failed' 
      };
    }
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['x-auth-token'];
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};

export default AuthContext;
```

### Task Board Component

```javascript
// frontend/src/components/TaskBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './TaskBoard.css';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`);
      
      const grouped = {
        todo: res.data.filter(t => t.status === 'todo'),
        'in-progress': res.data.filter(t => t.status === 'in-progress'),
        done: res.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (err) {
      console.error('Fetch tasks error:', err);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks();
    } catch (err) {
      console.error('Update task error:', err);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, newStatus);
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="task-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div 
          key={status}
          className="task-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{status.replace('-', ' ').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div
              key={task._id}
              className={`task-card priority-${task.priority}`}
              draggable
              onDragStart={(e) => handleDragStart(e, task._id)}
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <div className="task
