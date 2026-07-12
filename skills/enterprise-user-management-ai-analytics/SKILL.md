---
name: enterprise-user-management-ai-analytics
description: Enterprise user management system with AI-powered analytics for risk detection, task management, and support ticket automation
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management dashboard with AI insights"
  - "implement AI-powered ticket classification system"
  - "build task tracking with burnout detection"
  - "add risk prediction to user management"
  - "integrate ML analytics into enterprise system"
  - "configure AI-based anomaly detection for users"
  - "deploy full-stack user management with FastAPI ML"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application combining user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication, and user CRUD operations
- **Task Management**: Kanban boards, time tracking, and assignment workflows
- **Support Tickets**: AI-powered classification, routing, and resolution tracking
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay forecasting
- **Admin Dashboard**: Organization-wide metrics, audit logs, and security alerts

**Architecture**: React frontend + Node.js/Express backend + FastAPI ML service + MongoDB

## Installation

### Prerequisites

```bash
# Required
node >= 14.0.0
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
MONGODB_URI=mongodb://localhost:27017/enterprise_db
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
DATABASE_URL=mongodb://localhost:27017/enterprise_db
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
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
# Runs at http://localhost:3000
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
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    // Create user
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate token
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.status(201).json({ token, user: { id: user._id, username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
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
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
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
const auth = require('../middleware/auth');

// Get user tasks
router.get('/my-tasks', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('assignedBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching tasks', error: error.message });
  }
});

// Create task (admin only)
router.post('/', auth, async (req, res) => {
  try {
    if (req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Access denied' });
    }
    
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.userId,
      priority: priority || 'medium',
      dueDate,
      status: 'todo'
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error creating task', error: error.message });
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
    
    // Check authorization
    if (task.assignedTo.toString() !== req.user.userId && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Access denied' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error updating task', error: error.message });
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
import numpy as np
from datetime import datetime
import joblib
from sklearn.ensemble import RandomForestClassifier, IsolationForest
from river import anomaly, ensemble
import os

app = FastAPI(title="Enterprise AI Analytics Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
ticket_classifier = None
anomaly_detector = None
risk_predictor = None

# Pydantic models
class TicketRequest(BaseModel):
    title: str
    description: str
    priority: Optional[str] = "medium"

class UserBehaviorRequest(BaseModel):
    user_id: str
    login_count: int
    failed_logins: int
    tasks_completed: int
    avg_task_time: float
    tickets_raised: int
    login_hours: List[int]

class BurnoutRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_daily_hours: float
    missed_deadlines: int
    stress_indicators: int

class ProjectRequest(BaseModel):
    project_id: str
    total_tasks: int
    completed_tasks: int
    team_size: int
    avg_velocity: float
    days_remaining: int

@app.on_event("startup")
async def load_models():
    """Load or initialize ML models"""
    global ticket_classifier, anomaly_detector, risk_predictor
    
    model_path = os.getenv("MODEL_PATH", "./models")
    os.makedirs(model_path, exist_ok=True)
    
    # Initialize ticket classifier
    ticket_classifier = RandomForestClassifier(n_estimators=100, random_state=42)
    
    # Initialize anomaly detector (Isolation Forest)
    anomaly_detector = IsolationForest(contamination=0.1, random_state=42)
    
    # Initialize risk predictor
    risk_predictor = RandomForestClassifier(n_estimators=100, random_state=42)
    
    print("✅ ML models initialized")

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    """Classify support ticket and suggest department routing"""
    try:
        # Simple rule-based classification (extend with trained model)
        text = f"{request.title} {request.description}".lower()
        
        if any(word in text for word in ["password", "login", "access", "authenticate"]):
            category = "Security"
            department = "IT Security"
        elif any(word in text for word in ["bug", "error", "crash", "broken"]):
            category = "Technical"
            department = "Engineering"
        elif any(word in text for word in ["payment", "invoice", "billing"]):
            category = "Financial"
            department = "Finance"
        elif any(word in text for word in ["feature", "enhancement", "suggest"]):
            category = "Feature Request"
            department = "Product"
        else:
            category = "General"
            department = "Support"
        
        # Estimate resolution time based on priority and category
        resolution_time = {
            "high": {"Security": 2, "Technical": 4, "Financial": 6, "Feature Request": 12, "General": 8},
            "medium": {"Security": 4, "Technical": 8, "Financial": 12, "Feature Request": 24, "General": 16},
            "low": {"Security": 8, "Technical": 16, "Financial": 24, "Feature Request": 48, "General": 24}
        }
        
        estimated_hours = resolution_time.get(request.priority, {}).get(category, 12)
        
        return {
            "category": category,
            "department": department,
            "priority": request.priority,
            "estimated_resolution_hours": estimated_hours,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(request: UserBehaviorRequest):
    """Detect anomalous user behavior for security"""
    try:
        # Feature engineering
        features = np.array([[
            request.login_count,
            request.failed_logins,
            request.tasks_completed,
            request.avg_task_time,
            request.tickets_raised,
            len(set(request.login_hours)),  # Unique login hours
            max(request.login_hours) if request.login_hours else 0,
            min(request.login_hours) if request.login_hours else 0
        ]])
        
        # Simple anomaly detection logic
        is_anomaly = False
        risk_score = 0.0
        alerts = []
        
        # Failed login threshold
        if request.failed_logins > 5:
            is_anomaly = True
            risk_score += 0.3
            alerts.append("Excessive failed login attempts")
        
        # Unusual login hours (late night/early morning)
        unusual_hours = [h for h in request.login_hours if h < 6 or h > 22]
        if len(unusual_hours) > 3:
            is_anomaly = True
            risk_score += 0.2
            alerts.append("Unusual login times detected")
        
        # Low productivity
        if request.tasks_completed < 2 and request.login_count > 10:
            risk_score += 0.15
            alerts.append("Low task completion rate")
        
        # High ticket volume
        if request.tickets_raised > 10:
            risk_score += 0.1
            alerts.append("High support ticket volume")
        
        risk_score = min(risk_score, 1.0)
        
        return {
            "is_anomaly": is_anomaly,
            "risk_score": round(risk_score, 2),
            "alerts": alerts,
            "recommendation": "Monitor closely" if is_anomaly else "Normal behavior"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-burnout")
async def predict_burnout(request: BurnoutRequest):
    """Predict employee burnout risk"""
    try:
        # Calculate burnout indicators
        task_completion_rate = request.tasks_completed / max(request.tasks_assigned, 1)
        workload_score = request.tasks_assigned / max(request.team_size if hasattr(request, 'team_size') else 1, 1)
        
        burnout_score = 0.0
        risk_factors = []
        
        # High workload
        if request.avg_daily_hours > 9:
            burnout_score += 0.25
            risk_factors.append("Extended working hours")
        
        # Low completion rate
        if task_completion_rate < 0.7:
            burnout_score += 0.2
            risk_factors.append("Low task completion rate")
        
        # Missed deadlines
        if request.missed_deadlines > 3:
            burnout_score += 0.25
            risk_factors.append("Multiple missed deadlines")
        
        # Stress indicators
        if request.stress_indicators > 5:
            burnout_score += 0.3
            risk_factors.append("High stress indicators")
        
        burnout_score = min(burnout_score, 1.0)
        
        # Determine risk level
        if burnout_score > 0.7:
            risk_level = "high"
            recommendation = "Immediate intervention required - reduce workload, schedule check-in"
        elif burnout_score > 0.4:
            risk_level = "medium"
            recommendation = "Monitor closely - consider workload adjustment"
        else:
            risk_level = "low"
            recommendation = "Normal - maintain current support level"
        
        return {
            "burnout_score": round(burnout_score, 2),
            "risk_level": risk_level,
            "risk_factors": risk_factors,
            "recommendation": recommendation,
            "task_completion_rate": round(task_completion_rate, 2)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-project-delay")
async def predict_project_delay(request: ProjectRequest):
    """Predict project delay risk"""
    try:
        completion_rate = request.completed_tasks / max(request.total_tasks, 1)
        remaining_tasks = request.total_tasks - request.completed_tasks
        required_velocity = remaining_tasks / max(request.days_remaining, 1)
        
        delay_probability = 0.0
        risk_factors = []
        
        # Velocity check
        if required_velocity > request.avg_velocity * 1.5:
            delay_probability += 0.4
            risk_factors.append("Required velocity exceeds team capacity")
        
        # Completion rate check
        expected_completion = 1 - (request.days_remaining / 90)  # Assuming 90-day project
        if completion_rate < expected_completion - 0.2:
            delay_probability += 0.3
            risk_factors.append("Behind schedule")
        
        # Team size check
        if request.team_size < 3 and remaining_tasks > 20:
            delay_probability += 0.2
            risk_factors.append("Team size may be insufficient")
        
        delay_probability = min(delay_probability, 1.0)
        
        estimated_delay_days = 0
        if delay_probability > 0.5:
            estimated_delay_days = int((required_velocity - request.avg_velocity) * request.days_remaining)
        
        return {
            "delay_probability": round(delay_probability, 2),
            "estimated_delay_days": max(estimated_delay_days, 0),
            "completion_rate": round(completion_rate, 2),
            "required_velocity": round(required_velocity, 2),
            "current_velocity": request.avg_velocity,
            "risk_factors": risk_factors,
            "recommendation": "Increase resources or adjust timeline" if delay_probability > 0.6 else "On track"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "Enterprise AI Analytics"}
```

## Frontend React Components

### Task Kanban Board

```javascript
// frontend/src/components/TaskBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './TaskBoard.css';

const TaskBoard = () => {
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
      
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(categorized);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
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
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task, status }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        <span className="due-date">
          Due: {new Date(task.dueDate).toLocaleDateString()}
        </span>
      </div>
      <div className="task-actions">
        {status !== 'todo' && (
          <button onClick={() => updateTaskStatus(task._id, 'todo')}>← To Do</button>
        )}
        {status !== 'in-progress' && (
          <button onClick={() => updateTaskStatus(task._id, 'in-progress')}>
            ⏳ In Progress
          </button>
        )}
        {status !== 'done' && (
          <button onClick={() => updateTaskStatus(task._id, 'done')}>✓ Done</button>
        )}
      </div>
    </div>
  );

  if (loading) return <div className="loading">Loading tasks...</div>;

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => (
          <TaskCard key={task._id} task={task} status="todo" />
        ))}
      </div>
      
      <div className="kanban-column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => (
          <TaskCard key={task._id} task={task} status="in-progress" />
        ))}
      </div>
      
      <div className="kanban-column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => (
          <TaskCard key={task._id} task={task} status="done" />
        ))}
      </div>
    </div>
  );
};

export default TaskBoard;
```

### AI-Powered Support Ticket Form

```javascript
// frontend/src/components/TicketForm.js
import React, { useState } from 'react';
import axios from 'axios';

const TicketForm = () => {
  const [formData, setFormData] = useState({
    title: '',
    description: '',
    priority: 'medium'
  });
  const [aiSuggestion, setAiSuggestion] = useState(null);
  const [submitting, setSubmitting] = useState(false);

  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const getAISuggestion = async () => {
    if (!formData.title || !formData.description) return;
    
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_ML_URL}/api/ml/classify-ticket`,
        formData
      );
      setAiSuggestion(response.data);
    } catch (error) {
      console.error('Error getting AI suggestion:', error);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setSubmitting(true);
    
    try {
      const token = localStorage.getItem('token');
      await axios.post(
        `${process.env.REACT_APP_API_URL}/tickets`,
        {
          ...formData,
          category: aiSuggestion?.category,
          department: aiSuggestion?.department
        },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      alert('Ticket submitted successfully!');
      setFormData({ title: '', description: '', priority: 'medium' });
      setAiSuggestion(null);
    } catch (error) {
      console.error('Error submitting ticket:', error);
      alert('Failed to submit ticket');
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <div className="ticket-form">
      <h2>Create Support Ticket</h2>
      
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label>Title</label>
          <input
            type="text"
            name="title"
            value={formData.title}
            onChange={handleChange}
            onBlur={getAISuggestion}
            required
          />
        </div>
        
        <div className="form-group">
          <label>Description</label>
          <textarea
            name="description"
            value={formData.description}
            onChange={handleChange}
            onBlur={getAISuggestion}
            rows="5"
            required
          />
        </div>
        
        <div className="form-group">
          <label>Priority</label>
          <select name="priority" value={formData.priority} onChange={handleChange}>
            <option value="low">Low</option>
            <option value="medium">Medium</option>
            <option value="high">High</option>
          </select>
        </div>
        
        {aiSuggestion && (
          <div className="ai-suggestion">
            <h4>🤖 AI Analysis</h4>
            <p><strong>Category:</strong> {aiSuggestion.category}</p>
            <p><strong>Suggested Department:</strong> {aiSuggestion.department}</p>
            <p><strong>Estimated Resolution:</strong> {aiSuggestion.estimated_resolution_hours} hours</p>
            <p><strong>Confidence:</strong> {(aiSuggestion.confidence * 100).toFixed(0)}%</p>
          </div>
        )}
        
        <button type="submit" disabled={submitting}>
          {submitting ? 'Submitting...' : 'Submit Ticket'}
        </button>
      </form>
    </div>
  );
};

export default TicketForm;
```

## Database Models

### MongoDB User Model

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
    lowercase: true,
    trim: true
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
  department: {
    type: String,
    default: 'General'
  },
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: {
    type: Date
  },
  loginHistory: [{
    timestamp: Date,
    ipAddress: String,
    success: Boolean
  }],
  profile: {
    firstName: String,
    lastName: String,
    phone: String,
    avatar: String
  },
  preferences: {
    notifications: { type: Boolean, default: true },
    theme: { type: String, default: 'light' }
  }
}, {
  timestamps: true
});

module.exports = mongoose.model('User', userSchema);
```

### MongoDB Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
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
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: {
    type: Date,
    required: true
  },
  completedAt: {
    type: Date
  },
  timeTracking: {
    started: Date,
    elapsed: { type: Number, default: 0 } // in seconds
  },
  tags: [String],
  comments: [{
    user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
    text: String,
    timestamp: { type: Date, default: Date.now }
  }]
}, {
  timestamps: true
});

module.exports = mongoose.model('Task', taskSchema);
```

## Common Patterns

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

module.exports = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No token provided' });
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
// backend/middleware/authorize.js
module.exports = (...roles) => {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ message: 'Unauthorized' });
    }
    
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: 'Access denied' });
    }
    
    next();
  };
};

// Usage in routes
const authorize = require('../middleware/authorize');
router.post('/users', auth, authorize('admin'), createUser);
```

### Calling ML Service from Backend

```javascript
// backend/services/mlService.js
const axios = require('axios');

class MLService {
  constructor() {
    this.baseURL = process.env.ML_SERVICE_URL || 'http://localhost:8000';
  }

  async classifyTicket(title, description, priority) {
    try {
      const response = await axios.post(`${this.baseURL}/api/ml/classify-ticket`, {
        title,
        description,
        priority
      });
      return response.data;
    } catch (error) {
      console.error('ML Service Error:', error.message);
      return null;
    }
  }

  async detectAnomaly(userBehavior) {
    try {
      const response = await axios.post(`${this.baseURL}/api/ml/detect-anomaly`, userBehavior);
      return response.data;
    } catch (error) {
      console.error('ML Service Error:', error.message);
      return null;
    }
  }

  
