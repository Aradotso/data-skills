---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management capabilities
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "show me how to implement task tracking with AI insights"
  - "configure the user management system with ML service"
  - "build a user dashboard with burnout detection"
  - "create an admin panel with anomaly detection"
  - "implement JWT authentication for user management"
  - "set up AI-based ticket classification"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with AI-powered insights. It provides:

- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support System**: Ticket management with AI classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, security alerts

The system uses React.js frontend, Node.js backend, MongoDB database, and FastAPI-based ML service with scikit-learn and River for online learning.

## Installation

### Prerequisites

```bash
# Required: Node.js, Python 3.8+, MongoDB
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
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MODEL_PATH=./models
DATA_PATH=./data
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
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
# Runs at http://localhost:3000
```

## Key API Endpoints

### Authentication APIs

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register user
router.post('/register', async (req, res) => {
  const { username, email, password, role } = req.body;
  try {
    const user = await User.create({ username, email, password, role });
    const token = jwt.sign({ id: user._id, role: user.role }, 
                           process.env.JWT_SECRET, 
                           { expiresIn: process.env.JWT_EXPIRE });
    res.status(201).json({ success: true, token, user });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  try {
    const user = await User.findOne({ email }).select('+password');
    if (!user || !(await user.matchPassword(password))) {
      return res.status(401).json({ success: false, error: 'Invalid credentials' });
    }
    const token = jwt.sign({ id: user._id, role: user.role }, 
                           process.env.JWT_SECRET, 
                           { expiresIn: process.env.JWT_EXPIRE });
    res.json({ success: true, token, user });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

### User Management APIs

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');
const User = require('../models/User');

// Get all users (admin only)
router.get('/', protect, authorize('admin'), async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ success: true, count: users.length, data: users });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Get single user
router.get('/:id', protect, async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ success: false, error: 'User not found' });
    }
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Update user
router.put('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(req.params.id, req.body, {
      new: true,
      runValidators: true
    }).select('-password');
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Delete user
router.delete('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    await User.findByIdAndDelete(req.params.id);
    res.json({ success: true, data: {} });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

### Task Management APIs

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const { protect } = require('../middleware/auth');
const Task = require('../models/Task');

// Create task
router.post('/', protect, async (req, res) => {
  try {
    req.body.createdBy = req.user.id;
    const task = await Task.create(req.body);
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Get user tasks
router.get('/my-tasks', protect, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'username email')
      .sort('-createdAt');
    res.json({ success: true, count: tasks.length, data: tasks });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  const { status } = req.body; // 'todo', 'in-progress', 'done'
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true, runValidators: true }
    );
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Track time on task
router.post('/:id/time-log', protect, async (req, res) => {
  const { duration } = req.body; // in minutes
  try {
    const task = await Task.findById(req.params.id);
    task.timeSpent = (task.timeSpent || 0) + duration;
    await task.save();
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

### Ticket Management APIs

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const { protect } = require('../middleware/auth');
const Ticket = require('../models/Ticket');

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    req.body.createdBy = req.user.id;
    
    // Call ML service for classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      {
        title: req.body.title,
        description: req.body.description
      }
    );
    
    req.body.category = mlResponse.data.category;
    req.body.priority = mlResponse.data.priority;
    req.body.suggestedAssignee = mlResponse.data.suggested_assignee;
    
    const ticket = await Ticket.create(req.body);
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Get all tickets
router.get('/', protect, async (req, res) => {
  try {
    const query = req.user.role === 'admin' ? {} : { createdBy: req.user.id };
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .sort('-createdAt');
    res.json({ success: true, count: tickets.length, data: tickets });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

## ML Service APIs

### FastAPI ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from datetime import datetime

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    priority: str
    suggested_assignee: Optional[str]
    confidence: float

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_frequency: int
    failed_logins: int
    data_access_count: int
    unusual_hours_activity: int

class RiskPredictionResponse(BaseModel):
    risk_score: float
    risk_level: str
    factors: List[str]

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_pending: int
    avg_task_duration: float
    overtime_hours: float
    days_since_break: int

class BurnoutAnalysisResponse(BaseModel):
    burnout_score: float
    burnout_level: str
    recommendations: List[str]

# Ticket Classification
@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    """
    Classify support ticket using NLP and assign priority
    """
    try:
        # Load pre-trained model (simplified)
        text = f"{request.title} {request.description}".lower()
        
        # Category classification
        categories = {
            'technical': ['bug', 'error', 'crash', 'issue', 'broken'],
            'access': ['login', 'password', 'access', 'permission'],
            'feature': ['request', 'feature', 'enhancement', 'add'],
            'general': ['question', 'help', 'how', 'what']
        }
        
        category = 'general'
        max_matches = 0
        for cat, keywords in categories.items():
            matches = sum(1 for kw in keywords if kw in text)
            if matches > max_matches:
                max_matches = matches
                category = cat
        
        # Priority classification
        priority = 'medium'
        if any(word in text for word in ['urgent', 'critical', 'broken', 'down']):
            priority = 'high'
        elif any(word in text for word in ['minor', 'suggestion', 'enhancement']):
            priority = 'low'
        
        return TicketClassificationResponse(
            category=category,
            priority=priority,
            suggested_assignee=None,
            confidence=0.85
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Risk Prediction
@app.post("/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    """
    Predict security risk based on user behavior
    """
    try:
        # Calculate risk score (0-100)
        risk_factors = []
        risk_score = 0
        
        if request.failed_logins > 3:
            risk_score += 30
            risk_factors.append("Multiple failed login attempts")
        
        if request.unusual_hours_activity > 5:
            risk_score += 25
            risk_factors.append("Unusual activity hours")
        
        if request.data_access_count > 100:
            risk_score += 20
            risk_factors.append("High data access frequency")
        
        if request.login_frequency < 2:
            risk_score += 15
            risk_factors.append("Irregular login pattern")
        
        # Determine risk level
        if risk_score >= 70:
            risk_level = "high"
        elif risk_score >= 40:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return RiskPredictionResponse(
            risk_score=min(risk_score, 100),
            risk_level=risk_level,
            factors=risk_factors if risk_factors else ["No significant risk factors"]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout Detection
@app.post("/analyze-burnout", response_model=BurnoutAnalysisResponse)
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """
    Analyze employee burnout risk based on workload metrics
    """
    try:
        burnout_score = 0
        recommendations = []
        
        # Analyze workload
        if request.tasks_pending > 10:
            burnout_score += 25
            recommendations.append("Consider redistributing tasks")
        
        if request.avg_task_duration > 8:
            burnout_score += 20
            recommendations.append("Tasks taking longer than expected")
        
        if request.overtime_hours > 20:
            burnout_score += 30
            recommendations.append("Excessive overtime detected")
        
        if request.days_since_break > 60:
            burnout_score += 25
            recommendations.append("Schedule time off soon")
        
        # Determine burnout level
        if burnout_score >= 70:
            burnout_level = "high"
        elif burnout_score >= 40:
            burnout_level = "moderate"
        else:
            burnout_level = "low"
        
        if not recommendations:
            recommendations.append("Workload appears balanced")
        
        return BurnoutAnalysisResponse(
            burnout_score=min(burnout_score, 100),
            burnout_level=burnout_level,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Anomaly Detection
@app.post("/detect-anomaly")
async def detect_anomaly(data: dict):
    """
    Detect anomalous behavior in user activity
    """
    try:
        # Implement isolation forest or similar
        features = [
            data.get('login_time_deviation', 0),
            data.get('data_access_volume', 0),
            data.get('location_changes', 0),
            data.get('permission_requests', 0)
        ]
        
        # Simplified anomaly score
        anomaly_score = sum(abs(f) for f in features) / len(features)
        is_anomaly = anomaly_score > 2.5
        
        return {
            "is_anomaly": is_anomaly,
            "anomaly_score": anomaly_score,
            "timestamp": datetime.now().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Frontend Integration Examples

### Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

const API_URL = process.env.REACT_APP_API_URL;

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      loadUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const loadUser = async () => {
    try {
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };
      const res = await axios.get(`${API_URL}/api/auth/me`, config);
      setUser(res.data.data);
    } catch (error) {
      localStorage.removeItem('token');
      setToken(null);
    }
    setLoading(false);
  };

  const login = async (email, password) => {
    const res = await axios.post(`${API_URL}/api/auth/login`, { email, password });
    localStorage.setItem('token', res.data.token);
    setToken(res.data.token);
    setUser(res.data.user);
    return res.data;
  };

  const register = async (userData) => {
    const res = await axios.post(`${API_URL}/api/auth/register`, userData);
    localStorage.setItem('token', res.data.token);
    setToken(res.data.token);
    setUser(res.data.user);
    return res.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, token, login, register, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect, useContext } from 'react';
import axios from 'axios';
import { AuthContext } from '../context/AuthContext';
import './KanbanBoard.css';

const API_URL = process.env.REACT_APP_API_URL;

const KanbanBoard = () => {
  const { token } = useContext(AuthContext);
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const config = { headers: { Authorization: `Bearer ${token}` } };
      const res = await axios.get(`${API_URL}/api/tasks/my-tasks`, config);
      
      const categorized = {
        todo: res.data.data.filter(t => t.status === 'todo'),
        inProgress: res.data.data.filter(t => t.status === 'in-progress'),
        done: res.data.data.filter(t => t.status === 'done')
      };
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const config = { headers: { Authorization: `Bearer ${token}` } };
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        config
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task, status }) => (
    <div className="task-card" draggable onDragStart={(e) => e.dataTransfer.setData('taskId', task._id)}>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority-${task.priority}`}>{task.priority}</span>
        <span>Time: {task.timeSpent || 0}m</span>
      </div>
    </div>
  );

  const Column = ({ title, status, tasksList }) => (
    <div 
      className="kanban-column"
      onDragOver={(e) => e.preventDefault()}
      onDrop={(e) => {
        const taskId = e.dataTransfer.getData('taskId');
        updateTaskStatus(taskId, status);
      }}
    >
      <h3>{title} ({tasksList.length})</h3>
      <div className="task-list">
        {tasksList.map(task => (
          <TaskCard key={task._id} task={task} status={status} />
        ))}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <Column title="To Do" status="todo" tasksList={tasks.todo} />
      <Column title="In Progress" status="in-progress" tasksList={tasks.inProgress} />
      <Column title="Done" status="done" tasksList={tasks.done} />
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.js
import React, { useState, useEffect, useContext } from 'react';
import axios from 'axios';
import { AuthContext } from '../context/AuthContext';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;
const API_URL = process.env.REACT_APP_API_URL;

const AIAnalytics = () => {
  const { token, user } = useContext(AuthContext);
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const config = { headers: { Authorization: `Bearer ${token}` } };
      
      // Fetch user metrics
      const metricsRes = await axios.get(`${API_URL}/api/users/${user.id}/metrics`, config);
      const metrics = metricsRes.data.data;
      
      // Get risk prediction
      const riskRes = await axios.post(`${ML_API_URL}/predict-risk`, {
        user_id: user.id,
        login_frequency: metrics.loginFrequency,
        failed_logins: metrics.failedLogins,
        data_access_count: metrics.dataAccessCount,
        unusual_hours_activity: metrics.unusualHoursActivity
      });
      setRiskData(riskRes.data);
      
      // Get burnout analysis
      const burnoutRes = await axios.post(`${ML_API_URL}/analyze-burnout`, {
        user_id: user.id,
        tasks_completed: metrics.tasksCompleted,
        tasks_pending: metrics.tasksPending,
        avg_task_duration: metrics.avgTaskDuration,
        overtime_hours: metrics.overtimeHours,
        days_since_break: metrics.daysSinceBreak
      });
      setBurnoutData(burnoutRes.data);
      
      setLoading(false);
    } catch (error) {
      console.error('Error fetching analytics:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      <div className="analytics-grid">
        {/* Risk Assessment */}
        <div className="analytics-card">
          <h3>Security Risk Assessment</h3>
          <div className={`risk-indicator ${riskData.risk_level}`}>
            <span className="risk-score">{riskData.risk_score}</span>
            <span className="risk-level">{riskData.risk_level.toUpperCase()}</span>
          </div>
          <ul className="risk-factors">
            {riskData.factors.map((factor, idx) => (
              <li key={idx}>{factor}</li>
            ))}
          </ul>
        </div>
        
        {/* Burnout Analysis */}
        <div className="analytics-card">
          <h3>Burnout Analysis</h3>
          <div className={`burnout-indicator ${burnoutData.burnout_level}`}>
            <span className="burnout-score">{burnoutData.burnout_score}</span>
            <span className="burnout-level">{burnoutData.burnout_level.toUpperCase()}</span>
          </div>
          <div className="recommendations">
            <h4>Recommendations:</h4>
            <ul>
              {burnoutData.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        </div>
      </div>
      
      <button onClick={fetchAnalytics} className="refresh-btn">
        Refresh Analytics
      </button>
    </div>
  );
};

export default AIAnalytics;
```

### Ticket Creation with AI

```javascript
// frontend/src/components/CreateTicket.js
import React, { useState, useContext } from 'react';
import axios from 'axios';
import { AuthContext } from '../context/AuthContext';

const API_URL = process.env.REACT_APP_API_URL;

const CreateTicket = ({ onTicketCreated }) => {
  const { token } = useContext(AuthContext);
  const [formData, setFormData] = useState({
    title: '',
    description: ''
  });
  const [aiSuggestions, setAiSuggestions] = useState(null);
  const [loading, setLoading] = useState(false);

  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    
    try {
      const config = { headers: { Authorization: `Bearer ${token}` } };
      const res = await axios.post(`${API_URL}/api/tickets`, formData, config);
      
      // Show AI classification results
      setAiSuggestions({
        category: res.data.data.category,
        priority: res.data.data.priority,
        assignee: res.data.data.suggestedAssignee
      });
      
      setFormData({ title: '', description: '' });
      
      if (onTicketCreated) onTicketCreated(res.data.data);
      
      setTimeout(() => setAiSuggestions(null), 5000);
    } catch (error) {
      console.error('Error creating ticket:', error);
    }
    
    setLoading(false);
  };

  return (
    <div className="create-ticket">
      <h3>Create Support Ticket</h3>
      
      {aiSuggestions && (
        <div className="ai-suggestions">
          <h4>AI Classification:</h4>
          <p>Category: <strong>{aiSuggestions.category}</strong></p>
          <p>Priority: <strong>{aiSuggestions.priority}</strong></p>
          {aiSuggestions.assignee && (
            <p>Suggested Assignee: <strong>{aiSuggestions.assignee}</strong></p>
          )}
        </div>
      )}
      
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label>Title</label>
          <input
            type="text"
            name="title"
            value={formData.title}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-group">
          <label>Description</label>
          <textarea
            name="description"
            value={formData.description}
            onChange={handleChange}
            rows="5"
            required
          />
        </div>
        
        <button type="submit" disabled={loading}>
          {loading ? 'Creating...' : 'Create Ticket'}
        </button>
      </form>
    </div>
  );
};

export default CreateTicket;
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  username: {
    type: String,
    required: [true, 'Please provide username'],
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: [true, 'Please provide email'],
    unique: true,
    match: [/^\S+@\S+\.\S+$/, 'Please provide valid email']
  },
  password: {
    type: String,
    required: [true, 'Please provide password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: String,
  position: String,
  isActive: {
