---
name: enterprise-user-management-ai-system
description: Full-stack user management system with AI-powered analytics, risk detection, and task automation using React, Node.js, FastAPI, and MongoDB
triggers:
  - how do I set up the enterprise user management system
  - implement AI analytics for user behavior tracking
  - create a user management dashboard with risk detection
  - add AI-powered ticket classification to my app
  - build task management with burnout detection
  - integrate anomaly detection in user management
  - set up JWT authentication with role-based access
  - create kanban board with time tracking
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill helps you work with an enterprise-grade user management system that combines traditional CRUD operations with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights. The system uses React for frontend, Node.js/Express for backend, FastAPI for ML services, and MongoDB for data persistence.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Complete CRUD operations with role-based access control (Admin/User)
- **Task Management**: Kanban board with drag-and-drop, time tracking, and progress monitoring
- **Support Tickets**: Smart ticket system with AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay forecasting
- **Authentication**: JWT-based secure authentication with role management
- **Dashboards**: Separate admin and user dashboards with real-time insights

## Installation & Setup

### Prerequisites

```bash
# Required tools
node --version  # v14+ required
python --version  # 3.8+ required
mongod --version  # MongoDB 4.4+ required
```

### Backend Setup (Node.js)

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend server
npm start
```

### ML Service Setup (FastAPI)

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
DATABASE_URI=mongodb://localhost:27017/enterprise-user-mgmt
ENVIRONMENT=development
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Project Architecture

```
├── backend/           # Node.js/Express API
│   ├── models/       # MongoDB schemas
│   ├── routes/       # API endpoints
│   ├── middleware/   # Auth & validation
│   └── controllers/  # Business logic
├── ml-service/       # FastAPI ML service
│   ├── models/       # Trained ML models
│   ├── api/          # ML endpoints
│   └── utils/        # Feature engineering
└── frontend/         # React application
    ├── components/   # UI components
    ├── pages/        # Route pages
    └── services/     # API clients
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

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Hash password
    const hashedPassword = await bcrypt.hash(password, 10);
    
    // Create user
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate JWT
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );
    
    res.status(201).json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
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
    
    const isValidPassword = await bcrypt.compare(password, user.password);
    if (!isValidPassword) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );
    
    res.json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Task Management API

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
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
  timeSpent: { type: Number, default: 0 }, // in seconds
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { authenticateToken } = require('../middleware/auth');

const router = express.Router();

// Get all tasks (with filtering)
router.get('/', authenticateToken, async (req, res) => {
  try {
    const { status, assignedTo } = req.query;
    const filter = {};
    
    if (status) filter.status = status;
    if (assignedTo) filter.assignedTo = assignedTo;
    
    // Users see only their tasks, admins see all
    if (req.user.role !== 'admin') {
      filter.assignedTo = req.user.userId;
    }
    
    const tasks = await Task.find(filter)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task
router.post('/', authenticateToken, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.userId
    });
    
    await task.save();
    await task.populate('assignedTo', 'username email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
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
    
    task.status = status;
    task.updatedAt = Date.now();
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update time spent
router.patch('/:id/time', authenticateToken, async (req, res) => {
  try {
    const { timeSpent } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeSpent } },
      { new: true }
    );
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Support Ticket System

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { 
    type: String, 
    enum: ['technical', 'billing', 'general', 'urgent'],
    default: 'general'
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
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
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

```javascript
// backend/routes/tickets.js
const express = require('express');
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authenticateToken } = require('../middleware/auth');

const router = express.Router();

// Create ticket with AI classification
router.post('/', authenticateToken, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Call ML service for classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      { title, description }
    );
    
    const { category, priority, confidence, suggestedAssignee } = mlResponse.data;
    
    const ticket = new Ticket({
      title,
      description,
      category,
      priority,
      createdBy: req.user.userId,
      aiClassification: {
        category,
        confidence,
        suggestedAssignee
      }
    });
    
    await ticket.save();
    await ticket.populate('createdBy', 'username email');
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get all tickets
router.get('/', authenticateToken, async (req, res) => {
  try {
    const filter = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.userId };
    
    const tickets = await Ticket.find(filter)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service (FastAPI) Examples

### Ticket Classification

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import joblib
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer

app = FastAPI(title="Enterprise User Management ML Service")

# Load models (ensure these are trained beforehand)
try:
    ticket_classifier = joblib.load('models/ticket_classifier.pkl')
    vectorizer = joblib.load('models/ticket_vectorizer.pkl')
except:
    ticket_classifier = None
    vectorizer = None

class TicketRequest(BaseModel):
    title: str
    description: str

class TicketClassification(BaseModel):
    category: str
    priority: str
    confidence: float
    suggestedAssignee: Optional[str] = None

@app.post("/classify-ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketRequest):
    """Classify ticket using ML model"""
    try:
        # Combine title and description
        text = f"{ticket.title} {ticket.description}"
        
        if ticket_classifier is None or vectorizer is None:
            # Fallback to rule-based classification
            return rule_based_classification(ticket)
        
        # Vectorize text
        text_vector = vectorizer.transform([text])
        
        # Predict category and priority
        category = ticket_classifier.predict(text_vector)[0]
        confidence = float(np.max(ticket_classifier.predict_proba(text_vector)))
        
        # Determine priority based on keywords
        priority = determine_priority(text)
        
        return TicketClassification(
            category=category,
            priority=priority,
            confidence=confidence,
            suggestedAssignee=None  # Can be enhanced with user expertise matching
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def rule_based_classification(ticket: TicketRequest) -> TicketClassification:
    """Fallback rule-based classification"""
    text = f"{ticket.title} {ticket.description}".lower()
    
    # Category detection
    if any(word in text for word in ['bug', 'error', 'crash', 'not working']):
        category = 'technical'
    elif any(word in text for word in ['payment', 'invoice', 'billing', 'subscription']):
        category = 'billing'
    elif any(word in text for word in ['urgent', 'critical', 'immediately', 'asap']):
        category = 'urgent'
    else:
        category = 'general'
    
    priority = determine_priority(text)
    
    return TicketClassification(
        category=category,
        priority=priority,
        confidence=0.7,
        suggestedAssignee=None
    )

def determine_priority(text: str) -> str:
    """Determine priority based on text analysis"""
    text_lower = text.lower()
    
    critical_keywords = ['critical', 'urgent', 'emergency', 'down', 'outage']
    high_keywords = ['important', 'asap', 'quickly', 'soon']
    
    if any(word in text_lower for word in critical_keywords):
        return 'critical'
    elif any(word in text_lower for word in high_keywords):
        return 'high'
    elif len(text) > 500:  # Long descriptions might be complex
        return 'medium'
    else:
        return 'low'
```

### Risk Detection and Anomaly Detection

```python
# ml-service/api/analytics.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from typing import List, Dict
from datetime import datetime, timedelta
import numpy as np
from river import anomaly, tree

router = APIRouter(prefix="/analytics", tags=["analytics"])

# Online learning model for anomaly detection
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)

class UserBehavior(BaseModel):
    userId: str
    loginTime: datetime
    loginLocation: str
    tasksCompleted: int
    timeSpentHours: float
    failedLoginAttempts: int

class RiskScore(BaseModel):
    userId: str
    riskLevel: str  # low, medium, high, critical
    score: float
    factors: List[str]
    anomalyDetected: bool

@app.post("/analyze-risk", response_model=RiskScore)
async def analyze_risk(behavior: UserBehavior):
    """Analyze user behavior for risk assessment"""
    try:
        risk_factors = []
        score = 0.0
        
        # Failed login attempts
        if behavior.failedLoginAttempts > 3:
            score += 30
            risk_factors.append("Multiple failed login attempts")
        
        # Working hours analysis
        hour = behavior.loginTime.hour
        if hour < 6 or hour > 22:
            score += 15
            risk_factors.append("Unusual login time")
        
        # Task completion anomaly
        if behavior.tasksCompleted == 0 and behavior.timeSpentHours > 4:
            score += 20
            risk_factors.append("Low productivity detected")
        
        # Excessive work hours (burnout risk)
        if behavior.timeSpentHours > 12:
            score += 25
            risk_factors.append("Excessive work hours")
        
        # Anomaly detection using online learning
        features = {
            'hour': hour,
            'tasks_completed': behavior.tasksCompleted,
            'time_spent': behavior.timeSpentHours,
            'failed_logins': behavior.failedLoginAttempts
        }
        
        anomaly_score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        anomaly_detected = anomaly_score > 0.7
        if anomaly_detected:
            score += 35
            risk_factors.append("Behavioral anomaly detected")
        
        # Determine risk level
        if score >= 70:
            risk_level = "critical"
        elif score >= 50:
            risk_level = "high"
        elif score >= 30:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return RiskScore(
            userId=behavior.userId,
            riskLevel=risk_level,
            score=min(score, 100),
            factors=risk_factors,
            anomalyDetected=anomaly_detected
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Burnout Detection

```python
# ml-service/api/burnout.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from typing import List
from datetime import datetime

router = APIRouter(prefix="/burnout", tags=["burnout"])

class WorkloadData(BaseModel):
    userId: str
    weeklyHours: List[float]  # Hours worked each day
    tasksCompleted: int
    tasksAssigned: int
    averageTaskDuration: float
    overtimeHours: float

class BurnoutAnalysis(BaseModel):
    userId: str
    burnoutRisk: str  # low, moderate, high, severe
    score: float
    recommendations: List[str]
    alertAdmin: bool

@router.post("/detect", response_model=BurnoutAnalysis)
async def detect_burnout(workload: WorkloadData):
    """Detect burnout risk based on workload patterns"""
    try:
        score = 0.0
        recommendations = []
        
        # Weekly hours analysis
        avg_daily_hours = sum(workload.weeklyHours) / len(workload.weeklyHours)
        total_weekly_hours = sum(workload.weeklyHours)
        
        if total_weekly_hours > 50:
            score += 25
            recommendations.append("Reduce weekly working hours")
        
        if avg_daily_hours > 10:
            score += 20
            recommendations.append("Take regular breaks during the day")
        
        # Overtime analysis
        if workload.overtimeHours > 15:
            score += 30
            recommendations.append("Minimize overtime work")
        
        # Task completion rate
        completion_rate = workload.tasksCompleted / max(workload.tasksAssigned, 1)
        if completion_rate < 0.5:
            score += 15
            recommendations.append("Reassess task priorities and deadlines")
        
        # Task duration (possible indicator of struggle)
        if workload.averageTaskDuration > 8:  # hours
            score += 10
            recommendations.append("Break down complex tasks into smaller units")
        
        # Consistency check (variance in daily hours)
        if len(workload.weeklyHours) > 0:
            variance = np.var(workload.weeklyHours)
            if variance > 20:  # High variance
                score += 15
                recommendations.append("Maintain consistent work schedule")
        
        # Determine burnout risk
        if score >= 75:
            burnout_risk = "severe"
            alert_admin = True
            recommendations.append("URGENT: Schedule wellness check-in")
        elif score >= 50:
            burnout_risk = "high"
            alert_admin = True
            recommendations.append("Consider workload redistribution")
        elif score >= 30:
            burnout_risk = "moderate"
            alert_admin = False
            recommendations.append("Monitor workload closely")
        else:
            burnout_risk = "low"
            alert_admin = False
            recommendations.append("Continue current work patterns")
        
        return BurnoutAnalysis(
            userId=workload.userId,
            burnoutRisk=burnout_risk,
            score=min(score, 100),
            recommendations=recommendations,
            alertAdmin=alert_admin
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend React Examples

### Authentication Service

```javascript
// frontend/src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

class AuthService {
  async login(email, password) {
    const response = await axios.post(`${API_URL}/auth/login`, {
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    
    return response.data;
  }
  
  async register(username, email, password) {
    const response = await axios.post(`${API_URL}/auth/register`, {
      username,
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    
    return response.data;
  }
  
  logout() {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  }
  
  getCurrentUser() {
    const userStr = localStorage.getItem('user');
    return userStr ? JSON.parse(userStr) : null;
  }
  
  getToken() {
    return localStorage.getItem('token');
  }
  
  isAdmin() {
    const user = this.getCurrentUser();
    return user && user.role === 'admin';
  }
}

export default new AuthService();
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import authService from '../services/authService';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });
  
  const [draggedTask, setDraggedTask] = useState(null);
  
  useEffect(() => {
    fetchTasks();
  }, []);
  
  const fetchTasks = async () => {
    try {
      const token = authService.getToken();
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks`,
        {
          headers: { Authorization: `Bearer ${token}` }
        }
      );
      
      // Group tasks by status
      const grouped = {
        todo: [],
        'in-progress': [],
        done: []
      };
      
      response.data.forEach(task => {
        grouped[task.status].push(task);
      });
      
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };
  
  const handleDragStart = (task) => {
    setDraggedTask(task);
  };
  
  const handleDragOver = (e) => {
    e.preventDefault();
  };
  
  const handleDrop = async (status) => {
    if (!draggedTask) return;
    
    try {
      const token = authService.getToken();
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${draggedTask._id}/status`,
        { status },
        {
          headers: { Authorization: `Bearer ${token}` }
        }
      );
      
      fetchTasks();
      setDraggedTask(null);
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };
  
  const TaskCard = ({ task }) => (
    <div
      className="task-card"
      draggable
      onDragStart={() => handleDragStart(task)}
    >
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>
          {task.priority}
        </span>
        <span className="time-spent">
          {Math.floor(task.timeSpent / 3600)}h {Math.floor((task.timeSpent % 3600) / 60)}m
        </span>
      </div>
      {task.assignedTo && (
        <div className="assignee">
          Assigned to: {task.assignedTo.username}
        </div>
      )}
    </div>
  );
  
  return (
    <div className="kanban-board">
      {Object.entries(tasks).map(([status, taskList]) => (
        <div
          key={status}
          className="kanban-column"
          onDragOver={handleDragOver}
          onDrop={() => handleDrop(status)}
        >
          <h3 className="column-header">
            {status.replace('-', ' ').toUpperCase()}
            <span className="task-count">{taskList.length}</span>
          </h3>
          <div className="task-list">
            {taskList.map(task => (
              <TaskCard key={task._id} task={task} />
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Time Tracking Component

```javascript
// frontend/src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import authService from '../services/authService';
import './TimeTracker.css';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const [intervalId, setIntervalId] = useState(null);
  
  useEffect(() => {
    return () => {
      if (intervalId) {
        clearInterval(intervalId);
      }
    };
  }, [intervalId]);
  
  const startTimer = () => {
    setIsRunning(true);
    const id = setInterval(() => {
      setElapsedTime(prev => prev + 1);
    }, 1000);
    setIntervalId(id);
  };
  
  const stopTimer = async () => {
    if (intervalId) {
      clearInterval(intervalId);
    }
    setIsRunning(false);
    
    // Save time to backend
    try {
      const token = authService.getToken();
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`,
        { timeSpent: elapsedTime },
        {
          headers: { Authorization: `Bearer ${token}` }
        }
      );
      setElapsedTime(0);
    } catch (error) {
      console.error('Error saving time:', error);
    }
  };
  
  const formatTime = (seconds) => {
    const hours = Math.floor(seconds / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}:${String(secs).padStart(2, '0')}`;
  };
  
  return (
    <div className="time-tracker">
      <div className="timer-display">
        {formatTime(elapsedTime)}
      </div>
      <div className="timer-controls">
        {!isRunning ? (
          <button onClick={startTimer} className="btn-start">
            Start Timer
          </button>
        ) : (
          <button onClick={stopTimer} className="btn-stop">
            Stop & Save
          </button>
        )}
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
import authService from '../services/authService';
import './AdminDashboard.css';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeUsers: 0,
    totalTasks: 0,
    completedTasks: 0,
    openTickets: 0,
    highRiskUsers: []
  });
  
  const [burnoutAlerts, setBurnoutAlerts] = useState
