---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "add AI-powered risk detection to user system"
  - "build admin panel for user management"
  - "integrate ML service for burnout detection"
  - "configure JWT authentication for enterprise app"
  - "deploy user management system with AI features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack application for managing enterprise users, tasks, and support tickets with integrated AI/ML capabilities including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

## What This Project Does

- **User Management**: Centralized system for managing organizational users with role-based access control
- **Task Tracking**: Kanban-style board with time tracking and progress monitoring
- **Support Tickets**: AI-powered ticket classification and routing system
- **AI Analytics**: Machine learning models for risk detection, burnout prediction, and anomaly detection
- **Admin Dashboard**: Comprehensive analytics and audit logging for administrators
- **Authentication**: JWT-based secure authentication system

## Architecture

The system consists of three main components:

1. **Frontend** (React.js) - User and admin interfaces
2. **Backend** (Node.js) - REST API server with MongoDB
3. **ML Service** (FastAPI + scikit-learn) - AI/ML microservice

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
python >= 3.8
mongodb >= 4.4
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
EOF

# Start backend
npm start
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
    const { email, password, name, role } = req.body;
    
    const hashedPassword = await bcrypt.hash(password, 10);
    
    const user = new User({
      email,
      password: hashedPassword,
      name,
      role: role || 'user'
    });
    
    await user.save();
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ token, user: { id: user._id, email, name, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ token, user: { id: user._id, email, name: user.name, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

// Get user tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority, dueDate, assignedTo } = req.body;
    
    const task = new Task({
      title,
      description,
      priority: priority || 'medium',
      status: 'todo',
      dueDate,
      assignedTo: assignedTo || req.user.id,
      createdBy: req.user.id
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body; // 'todo', 'in-progress', 'done'
    
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time on task
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body; // in seconds
    
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { $inc: { timeSpent: duration } },
      { new: true }
    );
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Ticket API

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

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
      priority: priority || 'medium',
      status: 'open',
      category: mlResponse.data.category,
      suggestedAssignee: mlResponse.data.suggested_assignee,
      createdBy: req.user.id
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Admin: Get all tickets
router.get('/all', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find()
      .populate('createdBy', 'name email')
      .populate('assignedTo', 'name email')
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
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models (create if not exist)
MODEL_PATH = os.getenv("MODEL_PATH", "./models")

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    confidence: float
    suggested_assignee: Optional[str]

class RiskAnalysisRequest(BaseModel):
    user_id: str
    login_times: List[str]
    task_completion_rate: float
    failed_login_attempts: int
    access_patterns: List[dict]

class RiskAnalysisResponse(BaseModel):
    risk_score: float
    risk_level: str
    anomalies: List[str]
    recommendations: List[str]

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_pending: int
    avg_work_hours: float
    overtime_hours: float
    missed_deadlines: int

class BurnoutAnalysisResponse(BaseModel):
    burnout_score: float
    risk_level: str
    factors: dict
    recommendations: List[str]

@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket using AI"""
    try:
        # Simple rule-based classification (extend with ML model)
        text = f"{request.title} {request.description}".lower()
        
        categories = {
            "technical": ["bug", "error", "crash", "not working", "issue"],
            "access": ["login", "password", "access", "permission"],
            "feature": ["request", "enhancement", "suggestion", "add"],
            "general": ["question", "how", "help", "support"]
        }
        
        category = "general"
        max_score = 0
        
        for cat, keywords in categories.items():
            score = sum(1 for kw in keywords if kw in text)
            if score > max_score:
                max_score = score
                category = cat
        
        confidence = min(max_score / 3.0, 1.0)
        
        # Suggest assignee based on category
        assignees = {
            "technical": "tech_team",
            "access": "admin_team",
            "feature": "product_team",
            "general": "support_team"
        }
        
        return TicketClassificationResponse(
            category=category,
            confidence=confidence,
            suggested_assignee=assignees.get(category)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-risk", response_model=RiskAnalysisResponse)
async def analyze_risk(request: RiskAnalysisRequest):
    """Analyze user behavior for security risks"""
    try:
        anomalies = []
        risk_score = 0.0
        
        # Check failed login attempts
        if request.failed_login_attempts > 3:
            risk_score += 0.3
            anomalies.append("High failed login attempts")
        
        # Check task completion rate
        if request.task_completion_rate < 0.5:
            risk_score += 0.2
            anomalies.append("Low task completion rate")
        
        # Check login time patterns (unusual hours)
        unusual_hours = sum(1 for time_str in request.login_times 
                          if datetime.fromisoformat(time_str).hour < 6 or 
                          datetime.fromisoformat(time_str).hour > 22)
        
        if unusual_hours > len(request.login_times) * 0.3:
            risk_score += 0.25
            anomalies.append("Unusual login hours detected")
        
        # Check access patterns
        if len(request.access_patterns) > 100:  # Too many accesses
            risk_score += 0.25
            anomalies.append("Excessive access patterns")
        
        risk_level = "low"
        if risk_score > 0.7:
            risk_level = "high"
        elif risk_score > 0.4:
            risk_level = "medium"
        
        recommendations = []
        if risk_score > 0.5:
            recommendations.append("Enable two-factor authentication")
            recommendations.append("Review user permissions")
            recommendations.append("Monitor user activity closely")
        
        return RiskAnalysisResponse(
            risk_score=min(risk_score, 1.0),
            risk_level=risk_level,
            anomalies=anomalies,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout", response_model=BurnoutAnalysisResponse)
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze user workload for burnout risk"""
    try:
        burnout_score = 0.0
        factors = {}
        
        # Task load analysis
        total_tasks = request.tasks_completed + request.tasks_pending
        if total_tasks > 20:
            task_load_factor = min((total_tasks - 20) / 30, 1.0)
            burnout_score += task_load_factor * 0.3
            factors["task_overload"] = task_load_factor
        
        # Work hours analysis
        if request.avg_work_hours > 8:
            hours_factor = min((request.avg_work_hours - 8) / 4, 1.0)
            burnout_score += hours_factor * 0.3
            factors["excessive_hours"] = hours_factor
        
        # Overtime analysis
        if request.overtime_hours > 10:
            overtime_factor = min(request.overtime_hours / 20, 1.0)
            burnout_score += overtime_factor * 0.2
            factors["overtime"] = overtime_factor
        
        # Missed deadlines
        if request.missed_deadlines > 2:
            deadline_factor = min(request.missed_deadlines / 5, 1.0)
            burnout_score += deadline_factor * 0.2
            factors["missed_deadlines"] = deadline_factor
        
        risk_level = "low"
        if burnout_score > 0.7:
            risk_level = "high"
        elif burnout_score > 0.4:
            risk_level = "medium"
        
        recommendations = []
        if burnout_score > 0.5:
            recommendations.append("Reduce task assignments")
            recommendations.append("Encourage regular breaks")
            recommendations.append("Consider workload redistribution")
        if request.overtime_hours > 10:
            recommendations.append("Limit overtime hours")
        
        return BurnoutAnalysisResponse(
            burnout_score=min(burnout_score, 1.0),
            risk_level=risk_level,
            factors=factors,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Frontend React Components

### User Dashboard Component

```javascript
// frontend/src/components/UserDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [tickets, setTickets] = useState([]);
  const [loading, setLoading] = useState(true);
  const [activeTimer, setActiveTimer] = useState(null);
  const [timerSeconds, setTimerSeconds] = useState(0);

  useEffect(() => {
    fetchData();
  }, []);

  useEffect(() => {
    let interval;
    if (activeTimer) {
      interval = setInterval(() => {
        setTimerSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [activeTimer]);

  const fetchData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = { headers: { Authorization: `Bearer ${token}` } };

      const [tasksRes, ticketsRes] = await Promise.all([
        axios.get(`${API_URL}/tasks`, config),
        axios.get(`${API_URL}/tickets/my-tickets`, config)
      ]);

      // Group tasks by status
      const grouped = {
        todo: tasksRes.data.filter(t => t.status === 'todo'),
        inProgress: tasksRes.data.filter(t => t.status === 'in-progress'),
        done: tasksRes.data.filter(t => t.status === 'done')
      };

      setTasks(grouped);
      setTickets(ticketsRes.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching data:', error);
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${API_URL}/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const startTimer = (taskId) => {
    setActiveTimer(taskId);
    setTimerSeconds(0);
  };

  const stopTimer = async (taskId) => {
    try {
      const token = localStorage.getItem('token');
      await axios.post(
        `${API_URL}/tasks/${taskId}/time`,
        { duration: timerSeconds },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setActiveTimer(null);
      setTimerSeconds(0);
      fetchData();
    } catch (error) {
      console.error('Error saving time:', error);
    }
  };

  const formatTime = (seconds) => {
    const h = Math.floor(seconds / 3600);
    const m = Math.floor((seconds % 3600) / 60);
    const s = seconds % 60;
    return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  const TaskCard = ({ task, column }) => (
    <div className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority-${task.priority}`}>{task.priority}</span>
        <span>Time: {formatTime(task.timeSpent || 0)}</span>
      </div>
      
      {activeTimer === task._id ? (
        <div className="timer-active">
          <span>{formatTime(timerSeconds)}</span>
          <button onClick={() => stopTimer(task._id)}>Stop</button>
        </div>
      ) : (
        <button onClick={() => startTimer(task._id)}>Start Timer</button>
      )}

      <div className="task-actions">
        {column !== 'todo' && (
          <button onClick={() => updateTaskStatus(task._id, 'todo')}>← To Do</button>
        )}
        {column !== 'inProgress' && (
          <button onClick={() => updateTaskStatus(task._id, 'in-progress')}>
            {column === 'todo' ? '→' : '←'} In Progress
          </button>
        )}
        {column !== 'done' && (
          <button onClick={() => updateTaskStatus(task._id, 'done')}>→ Done</button>
        )}
      </div>
    </div>
  );

  if (loading) return <div>Loading...</div>;

  return (
    <div className="user-dashboard">
      <h1>My Dashboard</h1>
      
      <div className="kanban-board">
        <div className="kanban-column">
          <h2>To Do ({tasks.todo.length})</h2>
          {tasks.todo.map(task => (
            <TaskCard key={task._id} task={task} column="todo" />
          ))}
        </div>
        
        <div className="kanban-column">
          <h2>In Progress ({tasks.inProgress.length})</h2>
          {tasks.inProgress.map(task => (
            <TaskCard key={task._id} task={task} column="inProgress" />
          ))}
        </div>
        
        <div className="kanban-column">
          <h2>Done ({tasks.done.length})</h2>
          {tasks.done.map(task => (
            <TaskCard key={task._id} task={task} column="done" />
          ))}
        </div>
      </div>

      <div className="tickets-section">
        <h2>My Tickets</h2>
        {tickets.map(ticket => (
          <div key={ticket._id} className="ticket-card">
            <h4>{ticket.title}</h4>
            <span className={`status-${ticket.status}`}>{ticket.status}</span>
            <span className="category">{ticket.category}</span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component

```javascript
// frontend/src/components/AdminAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_URL = process.env.REACT_APP_ML_URL;

const AdminAnalytics = () => {
  const [users, setUsers] = useState([]);
  const [riskAnalysis, setRiskAnalysis] = useState({});
  const [burnoutAnalysis, setBurnoutAnalysis] = useState({});
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = { headers: { Authorization: `Bearer ${token}` } };

      const usersRes = await axios.get(`${API_URL}/admin/users`, config);
      setUsers(usersRes.data);

      // Fetch AI analytics for each user
      for (const user of usersRes.data) {
        await analyzeUserRisk(user);
        await analyzeBurnout(user);
      }

      setLoading(false);
    } catch (error) {
      console.error('Error fetching analytics:', error);
      setLoading(false);
    }
  };

  const analyzeUserRisk = async (user) => {
    try {
      const response = await axios.post(`${ML_URL}/analyze-risk`, {
        user_id: user._id,
        login_times: user.loginHistory || [],
        task_completion_rate: user.taskCompletionRate || 0.8,
        failed_login_attempts: user.failedLoginAttempts || 0,
        access_patterns: user.accessPatterns || []
      });

      setRiskAnalysis(prev => ({
        ...prev,
        [user._id]: response.data
      }));
    } catch (error) {
      console.error('Error analyzing risk:', error);
    }
  };

  const analyzeBurnout = async (user) => {
    try {
      const response = await axios.post(`${ML_URL}/analyze-burnout`, {
        user_id: user._id,
        tasks_completed: user.tasksCompleted || 0,
        tasks_pending: user.tasksPending || 0,
        avg_work_hours: user.avgWorkHours || 8,
        overtime_hours: user.overtimeHours || 0,
        missed_deadlines: user.missedDeadlines || 0
      });

      setBurnoutAnalysis(prev => ({
        ...prev,
        [user._id]: response.data
      }));
    } catch (error) {
      console.error('Error analyzing burnout:', error);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="admin-analytics">
      <h1>AI-Powered Analytics Dashboard</h1>

      <div className="analytics-grid">
        {users.map(user => (
          <div key={user._id} className="user-analytics-card">
            <h3>{user.name}</h3>
            <p>{user.email}</p>

            {riskAnalysis[user._id] && (
              <div className="risk-section">
                <h4>Security Risk Analysis</h4>
                <div className={`risk-level ${riskAnalysis[user._id].risk_level}`}>
                  Risk Level: {riskAnalysis[user._id].risk_level.toUpperCase()}
                </div>
                <div className="risk-score">
                  Score: {(riskAnalysis[user._id].risk_score * 100).toFixed(1)}%
                </div>
                {riskAnalysis[user._id].anomalies.length > 0 && (
                  <div className="anomalies">
                    <strong>Anomalies:</strong>
                    <ul>
                      {riskAnalysis[user._id].anomalies.map((anomaly, idx) => (
                        <li key={idx}>{anomaly}</li>
                      ))}
                    </ul>
                  </div>
                )}
              </div>
            )}

            {burnoutAnalysis[user._id] && (
              <div className="burnout-section">
                <h4>Burnout Risk Analysis</h4>
                <div className={`burnout-level ${burnoutAnalysis[user._id].risk_level}`}>
                  Burnout Risk: {burnoutAnalysis[user._id].risk_level.toUpperCase()}
                </div>
                <div className="burnout-score">
                  Score: {(burnoutAnalysis[user._id].burnout_score * 100).toFixed(1)}%
                </div>
                {burnoutAnalysis[user._id].recommendations.length > 0 && (
                  <div className="recommendations">
                    <strong>Recommendations:</strong>
                    <ul>
                      {burnoutAnalysis[user._id].recommendations.map((rec, idx) => (
                        <li key={idx}>{rec}</li>
                      ))}
                    </ul>
                  </div>
                )}
              </div>
            )}
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
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
  name: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  loginHistory: [{
    timestamp: Date,
    ipAddress: String
  }],
  failedLoginAttempts: {
    type: Number,
    default: 0
  },
  taskCompletionRate: {
    type: Number,
    default: 0
  },
  isActive: {
    type: Boolean,
    default: true
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

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
  
