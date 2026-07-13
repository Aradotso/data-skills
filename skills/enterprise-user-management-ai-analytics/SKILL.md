---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "build an enterprise user management system"
  - "integrate AI analytics for user behavior monitoring"
  - "create a task management system with Kanban board"
  - "implement JWT authentication for multi-role application"
  - "add AI-powered ticket classification and routing"
  - "set up burnout detection and anomaly analysis"
  - "build a user dashboard with time tracking"
  - "create an admin panel with role-based access control"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user management, task tracking, and support ticket systems with AI-powered insights. The system provides risk detection, anomaly analysis, burnout prediction, and intelligent ticket routing to improve organizational efficiency and decision-making.

**Key Components:**
- **Frontend**: React.js with Kanban board, time tracking, and dashboards
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI with scikit-learn and River for real-time predictions
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

```bash
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
```

Create `.env` file:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
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
# Frontend runs at http://localhost:3000
```

## Project Architecture

```
enterprise-user-management/
├── frontend/              # React.js application
│   ├── src/
│   │   ├── components/   # Reusable UI components
│   │   ├── pages/        # Page components
│   │   ├── contexts/     # React contexts (Auth, Theme)
│   │   ├── services/     # API service layers
│   │   └── utils/        # Helper functions
├── backend/              # Node.js API server
│   ├── models/           # MongoDB schemas
│   ├── routes/           # Express routes
│   ├── controllers/      # Business logic
│   ├── middleware/       # Auth, validation middleware
│   └── utils/            # Helper functions
└── ml-service/           # FastAPI ML service
    ├── models/           # ML model implementations
    ├── services/         # ML prediction services
    └── utils/            # Data processing utilities
```

## Core Features Implementation

### 1. Authentication & Authorization

#### Backend: JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      throw new Error('No authentication token provided');
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findOne({ _id: decoded._id });

    if (!user) {
      throw new Error('User not found');
    }

    req.token = token;
    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
  }
};

const adminAuth = async (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { auth, adminAuth };
```

#### Frontend: Authentication Service

```javascript
// frontend/src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

class AuthService {
  async login(email, password) {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('user', JSON.stringify(response.data));
    }
    
    return response.data;
  }

  logout() {
    localStorage.removeItem('user');
  }

  getCurrentUser() {
    return JSON.parse(localStorage.getItem('user'));
  }

  getAuthHeader() {
    const user = this.getCurrentUser();
    if (user && user.token) {
      return { Authorization: `Bearer ${user.token}` };
    }
    return {};
  }
}

export default new AuthService();
```

### 2. User Management

#### Backend: User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
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
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  position: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

userSchema.methods.comparePassword = async function(password) {
  return await bcrypt.compare(password, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

#### Backend: User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');

exports.getAllUsers = async (req, res) => {
  try {
    const { page = 1, limit = 10, role, status } = req.query;
    const query = {};
    
    if (role) query.role = role;
    if (status) query.status = status;

    const users = await User.find(query)
      .select('-password')
      .limit(limit * 1)
      .skip((page - 1) * limit)
      .sort({ createdAt: -1 });

    const count = await User.countDocuments(query);

    res.json({
      users,
      totalPages: Math.ceil(count / limit),
      currentPage: page,
      total: count
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.createUser = async (req, res) => {
  try {
    const { name, email, password, role, department, position } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'Email already exists' });
    }

    const user = new User({
      name,
      email,
      password,
      role,
      department,
      position
    });

    await user.save();

    res.status(201).json({
      message: 'User created successfully',
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;

    delete updates.password; // Prevent password update through this route

    const user = await User.findByIdAndUpdate(
      id,
      updates,
      { new: true, runValidators: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json({ message: 'User updated successfully', user });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.deleteUser = async (req, res) => {
  try {
    const { id } = req.params;
    const user = await User.findByIdAndDelete(id);

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

### 3. Task Management with Kanban Board

#### Backend: Task Model

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
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
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
  timeTracked: {
    type: Number,
    default: 0 // in seconds
  },
  startedAt: Date,
  completedAt: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', taskSchema);
```

#### Frontend: Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import authService from '../services/authService';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    inprogress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks`,
        { headers: authService.getAuthHeader() }
      );
      
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inprogress: response.data.filter(t => t.status === 'inprogress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`,
        { status: newStatus },
        { headers: authService.getAuthHeader() }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragStart = (e, taskId, status) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('fromStatus', status);
  };

  const handleDrop = (e, toStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const fromStatus = e.dataTransfer.getData('fromStatus');
    
    if (fromStatus !== toStatus) {
      updateTaskStatus(taskId, toStatus);
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
            onDragStart={(e) => handleDragStart(e, task._id, status)}
          >
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <div className="task-meta">
              <span className={`priority ${task.priority}`}>
                {task.priority}
              </span>
              {task.dueDate && (
                <span className="due-date">
                  Due: {new Date(task.dueDate).toLocaleDateString()}
                </span>
              )}
            </div>
          </div>
        ))}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do')}
      {renderColumn('inprogress', 'In Progress')}
      {renderColumn('done', 'Done')}
    </div>
  );
};

export default KanbanBoard;
```

### 4. Time Tracking

#### Frontend: Time Tracker Component

```javascript
// frontend/src/components/TimeTracker.jsx
import React, { useState, useEffect, useRef } from 'react';
import axios from 'axios';
import authService from '../services/authService';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setElapsedTime(prev => prev + 1);
      }, 1000);
    } else if (intervalRef.current) {
      clearInterval(intervalRef.current);
    }

    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    };
  }, [isRunning]);

  const formatTime = (seconds) => {
    const hours = Math.floor(seconds / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const handleStart = () => {
    setIsRunning(true);
  };

  const handlePause = () => {
    setIsRunning(false);
  };

  const handleStop = async () => {
    setIsRunning(false);
    
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`,
        { timeTracked: elapsedTime },
        { headers: authService.getAuthHeader() }
      );
      
      setElapsedTime(0);
    } catch (error) {
      console.error('Error saving time:', error);
    }
  };

  return (
    <div className="time-tracker">
      <div className="time-display">
        {formatTime(elapsedTime)}
      </div>
      <div className="time-controls">
        {!isRunning ? (
          <button onClick={handleStart} className="btn-start">
            Start
          </button>
        ) : (
          <button onClick={handlePause} className="btn-pause">
            Pause
          </button>
        )}
        <button onClick={handleStop} className="btn-stop">
          Stop & Save
        </button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

### 5. Support Ticket System

#### Backend: Ticket Model

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
    enum: ['technical', 'access', 'general', 'bug', 'feature'],
    required: true
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
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
    suggestedAssignee: mongoose.Schema.Types.ObjectId
  },
  comments: [{
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User'
    },
    message: String,
    createdAt: {
      type: Date,
      default: Date.now
    }
  }],
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### 6. AI-Powered Features

#### ML Service: Risk Detection

```python
# ml-service/services/risk_detection.py
from fastapi import APIRouter
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import IsolationForest
from datetime import datetime, timedelta

router = APIRouter()

class UserBehavior(BaseModel):
    user_id: str
    login_frequency: int
    failed_login_attempts: int
    access_outside_hours: int
    data_download_volume: float
    permission_changes: int
    unusual_location_access: int

class RiskPrediction(BaseModel):
    user_id: str
    risk_score: float
    risk_level: str
    factors: List[str]
    timestamp: str

# Initialize model (in production, load from saved model)
risk_model = IsolationForest(contamination=0.1, random_state=42)

@router.post("/predict-risk", response_model=RiskPrediction)
async def predict_risk(behavior: UserBehavior):
    """
    Predict user risk based on behavior patterns
    """
    # Prepare features
    features = np.array([[
        behavior.login_frequency,
        behavior.failed_login_attempts,
        behavior.access_outside_hours,
        behavior.data_download_volume,
        behavior.permission_changes,
        behavior.unusual_location_access
    ]])
    
    # Get anomaly score (-1 for anomaly, 1 for normal)
    prediction = risk_model.fit_predict(features)
    anomaly_score = risk_model.score_samples(features)[0]
    
    # Convert to risk score (0-100)
    risk_score = float(max(0, min(100, (1 - anomaly_score) * 50)))
    
    # Determine risk level
    if risk_score >= 75:
        risk_level = "critical"
    elif risk_score >= 50:
        risk_level = "high"
    elif risk_score >= 25:
        risk_level = "medium"
    else:
        risk_level = "low"
    
    # Identify risk factors
    factors = []
    if behavior.failed_login_attempts > 3:
        factors.append("Multiple failed login attempts")
    if behavior.access_outside_hours > 5:
        factors.append("Frequent access outside business hours")
    if behavior.data_download_volume > 1000:
        factors.append("High data download volume")
    if behavior.permission_changes > 2:
        factors.append("Unusual permission changes")
    if behavior.unusual_location_access > 0:
        factors.append("Access from unusual locations")
    
    return RiskPrediction(
        user_id=behavior.user_id,
        risk_score=risk_score,
        risk_level=risk_level,
        factors=factors if factors else ["No significant risk factors"],
        timestamp=datetime.now().isoformat()
    )
```

#### ML Service: Burnout Detection

```python
# ml-service/services/burnout_detection.py
from fastapi import APIRouter
from pydantic import BaseModel
from typing import Optional
import numpy as np

router = APIRouter()

class WorkloadMetrics(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_completion_time: float  # in hours
    overtime_hours: float
    weekend_work_hours: float
    missed_deadlines: int
    days_without_break: int

class BurnoutPrediction(BaseModel):
    user_id: str
    burnout_score: float
    risk_level: str
    recommendations: list
    warning_signs: list

@router.post("/detect-burnout", response_model=BurnoutPrediction)
async def detect_burnout(metrics: WorkloadMetrics):
    """
    Detect potential burnout based on workload metrics
    """
    # Calculate individual risk factors (0-100 scale)
    task_overload = min(100, (metrics.tasks_assigned / 50) * 100)
    completion_rate = (metrics.tasks_completed / max(1, metrics.tasks_assigned)) * 100
    completion_rate_risk = 100 - completion_rate
    
    overtime_risk = min(100, (metrics.overtime_hours / 20) * 100)
    weekend_risk = min(100, (metrics.weekend_work_hours / 16) * 100)
    deadline_risk = min(100, (metrics.missed_deadlines / 5) * 100)
    rest_risk = min(100, (metrics.days_without_break / 14) * 100)
    
    # Weighted burnout score
    burnout_score = (
        task_overload * 0.15 +
        completion_rate_risk * 0.20 +
        overtime_risk * 0.20 +
        weekend_risk * 0.15 +
        deadline_risk * 0.15 +
        rest_risk * 0.15
    )
    
    # Determine risk level
    if burnout_score >= 70:
        risk_level = "critical"
    elif burnout_score >= 50:
        risk_level = "high"
    elif burnout_score >= 30:
        risk_level = "moderate"
    else:
        risk_level = "low"
    
    # Generate warning signs
    warning_signs = []
    if task_overload > 70:
        warning_signs.append("Excessive task load")
    if completion_rate < 60:
        warning_signs.append("Low task completion rate")
    if overtime_risk > 60:
        warning_signs.append("Consistent overtime work")
    if weekend_risk > 50:
        warning_signs.append("Frequent weekend work")
    if deadline_risk > 40:
        warning_signs.append("Multiple missed deadlines")
    if rest_risk > 60:
        warning_signs.append("Insufficient rest periods")
    
    # Generate recommendations
    recommendations = []
    if burnout_score >= 50:
        recommendations.extend([
            "Consider redistributing workload",
            "Schedule mandatory time off",
            "Review project deadlines and priorities"
        ])
    if overtime_risk > 50:
        recommendations.append("Limit overtime to essential tasks only")
    if rest_risk > 50:
        recommendations.append("Ensure regular breaks and time off")
    if completion_rate < 70:
        recommendations.append("Provide additional support or training")
    
    if not recommendations:
        recommendations.append("Maintain current work-life balance")
    
    return BurnoutPrediction(
        user_id=metrics.user_id,
        burnout_score=round(burnout_score, 2),
        risk_level=risk_level,
        recommendations=recommendations,
        warning_signs=warning_signs if warning_signs else ["No immediate concerns"]
    )
```

#### ML Service: Ticket Classification

```python
# ml-service/services/ticket_classification.py
from fastapi import APIRouter
from pydantic import BaseModel
from typing import Optional
import re
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB

router = APIRouter()

class TicketData(BaseModel):
    title: str
    description: str

class TicketClassification(BaseModel):
    category: str
    confidence: float
    suggested_priority: str
    keywords: list

# Simple keyword-based classification (in production, use trained ML model)
CATEGORY_KEYWORDS = {
    'technical': ['error', 'bug', 'crash', 'not working', 'failed', 'broken', 'issue'],
    'access': ['password', 'login', 'permission', 'access', 'unlock', 'account'],
    'feature': ['feature', 'enhancement', 'improve', 'add', 'new', 'request'],
    'general': ['question', 'how to', 'help', 'information', 'query']
}

PRIORITY_KEYWORDS = {
    'critical': ['urgent', 'critical', 'emergency', 'immediately', 'asap', 'production'],
    'high': ['important', 'soon', 'blocking', 'cannot', 'unable'],
    'medium': ['should', 'would', 'need', 'want'],
    'low': ['minor', 'eventually', 'nice to have', 'suggestion']
}

@router.post("/classify-ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketData):
    """
    Classify support ticket category and priority
    """
    text = f"{ticket.title} {ticket.description}".lower()
    
    # Calculate category scores
    category_scores = {}
    for category, keywords in CATEGORY_KEYWORDS.items():
        score = sum(1 for keyword in keywords if keyword in text)
        category_scores[category] = score
    
    # Determine category
    category = max(category_scores, key=category_scores.get)
    if category_scores[category] == 0:
        category = 'general'
    
    confidence = min(100, (category_scores[category] / len(CATEGORY_KEYWORDS[category])) * 100)
    
    # Determine priority
    priority_scores = {}
    for priority, keywords in PRIORITY_KEYWORDS.items():
        score = sum(1 for keyword in keywords if keyword in text)
        priority_scores[priority] = score
    
    suggested_priority = max(priority_scores, key=priority_scores.get)
    if priority_scores[suggested_priority] == 0:
        suggested_priority = 'medium'
    
    # Extract keywords
    words = re.findall(r'\b\w+\b', text)
    keywords = [w for w in words if len(w) > 4][:5]
    
    return TicketClassification(
        category=category,
        confidence=round(confidence, 2),
        suggested_priority=suggested_priority,
        keywords=keywords
    )
```

#### Backend: Integrating AI Services

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL || 'http://localhost:8000';

exports.createTicket = async (req, res) => {
  try {
    const { title, description, category } = req.body;
    
    // Get AI classification
    let aiClassification = null;
    try {
      const classificationResponse = await axios.post(
        `${ML_SERVICE_URL}/classify-ticket`,
        { title, description }
      );
      aiClassification = classificationResponse.data;
    } catch (error) {
      console.error('AI classification failed:', error.message);
    }

    const ticket = new Ticket({
      title,
      description,
      category: category || aiClassification?.category || 'general',
      priority: aiClassification?.suggested_priority || 'medium',
      createdBy: req.user._id,
      aiClassification
    });

    await ticket.save();

    res.status(201).json({
      message: 'Ticket created successfully',
      ticket,
      aiInsights: aiClassification
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.getUserRiskProfile = async (req, res) => {
  try {
    const { userId } = req.params;
    
    // Gather user behavior metrics (example data)
    const behaviorMetrics = {
      user_id: userId,
      login_frequency: 45,
      failed_login_attempts: 1,
      access_outside_hours: 3,
      data_download_volume: 250.5,
      permission_changes: 0,
      unusual_location_access: 0
    };

    const riskResponse = await axios.post(
      `${ML_SERVICE_URL}/predict-risk`,
      behaviorMetrics
    );

    res.json(riskResponse.data);
  } catch (error) {
    res.status(500).json({ error:
