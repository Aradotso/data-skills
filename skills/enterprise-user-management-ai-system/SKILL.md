---
name: enterprise-user-management-ai-system
description: Full-stack user management system with AI-powered analytics, risk detection, and task management for enterprise organizations
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with risk detection"
  - "build task management system with AI insights"
  - "integrate AI analytics for user behavior"
  - "deploy enterprise user management with ML"
  - "configure AI-driven ticket classification"
  - "implement burnout detection for users"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack application that combines user management, task tracking, and support ticket systems with AI-powered insights. It provides risk detection, anomaly detection, burnout analysis, and predictive project insights using machine learning models.

**Key Components:**
- **Frontend**: React.js dashboard for users and admins
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based AI engine with scikit-learn and River
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

```bash
# Required software
node -v  # Node.js 14+
python --version  # Python 3.8+
mongod --version  # MongoDB 4.4+
```

### Clone and Setup

```bash
# Clone repository
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
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
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
LOG_LEVEL=INFO
API_PORT=8000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
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
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Architecture

### Backend API Structure (Node.js)

```javascript
// server.js - Main entry point
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Connect to MongoDB
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
});

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
// middleware/auth.js
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

### User Model

```javascript
// models/User.js
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
  tasksCompleted: {
    type: Number,
    default: 0
  },
  hoursWorked: {
    type: Number,
    default: 0
  },
  ticketsRaised: {
    type: Number,
    default: 0
  },
  riskScore: {
    type: Number,
    default: 0
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
// models/Task.js
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
    default: 0
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
// models/Ticket.js
const mongoose = require('mongoose');

const TicketSchema = new mongoose.Schema({
  subject: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  raisedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  category: {
    type: String,
    enum: ['technical', 'hr', 'admin', 'other']
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  aiClassification: Object,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Ticket', TicketSchema);
```

## API Endpoints

### Authentication

```javascript
// routes/auth.js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// @route   POST /api/auth/register
// @desc    Register user
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;
    
    let user = await User.findOne({ email });
    if (user) {
      return res.status(400).json({ msg: 'User already exists' });
    }
    
    user = new User({
      name,
      email,
      password,
      role,
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
        res.json({ token, user: { id: user.id, name: user.name, role: user.role } });
      }
    );
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// @route   POST /api/auth/login
// @desc    Authenticate user & get token
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

module.exports = router;
```

### Task Management

```javascript
// routes/tasks.js
const express = require('express');
const router = express.Router();
const auth = require('../middleware/auth');
const Task = require('../models/Task');

// @route   GET /api/tasks
// @desc    Get all tasks for user
router.get('/', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// @route   POST /api/tasks
// @desc    Create task (admin only)
router.post('/', auth, async (req, res) => {
  try {
    if (req.user.role !== 'admin') {
      return res.status(403).json({ msg: 'Not authorized' });
    }
    
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority,
      dueDate
    });
    
    await task.save();
    res.json(task);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// @route   PUT /api/tasks/:id
// @desc    Update task status
router.put('/:id', auth, async (req, res) => {
  try {
    const { status, timeSpent } = req.body;
    
    let task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ msg: 'Task not found' });
    }
    
    // Check if user owns task or is admin
    if (task.assignedTo.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ msg: 'Not authorized' });
    }
    
    task.status = status || task.status;
    if (timeSpent) task.timeSpent += timeSpent;
    
    await task.save();
    res.json(task);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

module.exports = router;
```

### Ticket Management with AI Integration

```javascript
// routes/tickets.js
const express = require('express');
const router = express.Router();
const auth = require('../middleware/auth');
const axios = require('axios');
const Ticket = require('../models/Ticket');

// @route   POST /api/tickets
// @desc    Create support ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { subject, description } = req.body;
    
    // Call ML service for AI classification
    let aiClassification = null;
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/classify-ticket`,
        { subject, description }
      );
      aiClassification = mlResponse.data;
    } catch (mlErr) {
      console.error('ML service error:', mlErr.message);
    }
    
    const ticket = new Ticket({
      subject,
      description,
      raisedBy: req.user.id,
      category: aiClassification?.category || 'other',
      priority: aiClassification?.priority || 'medium',
      aiClassification
    });
    
    await ticket.save();
    res.json(ticket);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

// @route   GET /api/tickets
// @desc    Get tickets
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

### Main ML Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, tree
import joblib
import os

app = FastAPI()

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models directory
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Anomaly detector (online learning)
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    tasks_completed: int
    hours_worked: float
    tickets_raised: int
    days_since_joined: int

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    hours_worked: float
    tasks_pending: int
    overtime_hours: float
    days_without_break: int

@app.get("/")
def read_root():
    return {"message": "Enterprise User Management ML Service", "status": "running"}

@app.post("/api/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """
    AI-powered ticket classification
    """
    try:
        text = f"{request.subject} {request.description}".lower()
        
        # Simple keyword-based classification (can be replaced with NLP model)
        category_keywords = {
            'technical': ['bug', 'error', 'crash', 'not working', 'broken', 'issue'],
            'hr': ['leave', 'salary', 'payroll', 'vacation', 'policy'],
            'admin': ['access', 'permission', 'account', 'login', 'password']
        }
        
        category_scores = {}
        for cat, keywords in category_keywords.items():
            score = sum(1 for keyword in keywords if keyword in text)
            category_scores[cat] = score
        
        category = max(category_scores, key=category_scores.get) if max(category_scores.values()) > 0 else 'other'
        
        # Priority based on urgency keywords
        urgency_keywords = ['urgent', 'critical', 'emergency', 'asap', 'immediately']
        priority = 'high' if any(word in text for word in urgency_keywords) else 'medium'
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85,
            "keywords_matched": category_scores
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """
    Predict user risk score based on behavior patterns
    """
    try:
        # Feature engineering
        completion_rate = request.tasks_completed / max(request.days_since_joined / 7, 1)
        hours_per_task = request.hours_worked / max(request.tasks_completed, 1)
        ticket_frequency = request.tickets_raised / max(request.days_since_joined / 30, 1)
        
        # Simple risk scoring (can be replaced with trained model)
        risk_score = 0
        
        # Low completion rate increases risk
        if completion_rate < 2:
            risk_score += 30
        
        # Too many hours per task
        if hours_per_task > 10:
            risk_score += 25
        
        # High ticket frequency
        if ticket_frequency > 3:
            risk_score += 20
        
        # Recent activity
        if request.days_since_joined < 30:
            risk_score += 15
        
        risk_score = min(risk_score, 100)
        
        risk_level = 'high' if risk_score > 70 else 'medium' if risk_score > 40 else 'low'
        
        return {
            "user_id": request.user_id,
            "risk_score": risk_score,
            "risk_level": risk_level,
            "factors": {
                "completion_rate": completion_rate,
                "hours_per_task": hours_per_task,
                "ticket_frequency": ticket_frequency
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    """
    Detect potential employee burnout using workload metrics
    """
    try:
        # Calculate burnout indicators
        burnout_score = 0
        indicators = []
        
        # Excessive hours
        if request.hours_worked > 50:
            burnout_score += 30
            indicators.append("Excessive working hours")
        
        # High task load
        if request.tasks_pending > 10:
            burnout_score += 25
            indicators.append("High number of pending tasks")
        
        # Overtime
        if request.overtime_hours > 10:
            burnout_score += 25
            indicators.append("Frequent overtime")
        
        # No breaks
        if request.days_without_break > 14:
            burnout_score += 20
            indicators.append("Extended period without breaks")
        
        burnout_score = min(burnout_score, 100)
        risk_level = 'high' if burnout_score > 70 else 'medium' if burnout_score > 40 else 'low'
        
        recommendations = []
        if burnout_score > 40:
            recommendations.append("Schedule time off")
            recommendations.append("Redistribute workload")
            recommendations.append("Encourage regular breaks")
        
        return {
            "user_id": request.user_id,
            "burnout_score": burnout_score,
            "risk_level": risk_level,
            "indicators": indicators,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/detect-anomaly")
async def detect_anomaly(data: dict):
    """
    Detect anomalous user behavior using online learning
    """
    try:
        # Extract features
        features = {
            'login_hour': data.get('login_hour', 9),
            'session_duration': data.get('session_duration', 8),
            'api_calls': data.get('api_calls', 100),
            'failed_attempts': data.get('failed_attempts', 0)
        }
        
        # Learn and score
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.7
        
        return {
            "anomaly_score": float(score),
            "is_anomaly": is_anomaly,
            "threshold": 0.7,
            "features": features
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Frontend React Components

### User Dashboard

```javascript
// frontend/src/components/UserDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [tickets, setTickets] = useState([]);
  const [stats, setStats] = useState({});
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = {
        headers: { 'x-auth-token': token }
      };

      const [tasksRes, ticketsRes, statsRes] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/api/tickets`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/api/analytics/user-stats`, config)
      ]);

      setTasks(tasksRes.data);
      setTickets(ticketsRes.data);
      setStats(statsRes.data);
      setLoading(false);
    } catch (err) {
      console.error(err);
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.put(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`,
        { status: newStatus },
        { headers: { 'x-auth-token': token } }
      );
      fetchDashboardData();
    } catch (err) {
      console.error(err);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Tasks Completed</h3>
          <p>{stats.tasksCompleted || 0}</p>
        </div>
        <div className="stat-card">
          <h3>Hours Worked</h3>
          <p>{stats.hoursWorked || 0}</p>
        </div>
        <div className="stat-card">
          <h3>Tickets Raised</h3>
          <p>{stats.ticketsRaised || 0}</p>
        </div>
        <div className="stat-card">
          <h3>Risk Score</h3>
          <p className={stats.riskScore > 70 ? 'high-risk' : 'low-risk'}>
            {stats.riskScore || 0}
          </p>
        </div>
      </div>

      <div className="kanban-board">
        <h2>My Tasks</h2>
        <div className="kanban-columns">
          <div className="kanban-column">
            <h3>To Do</h3>
            {tasks.filter(t => t.status === 'todo').map(task => (
              <div key={task._id} className="task-card">
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <button onClick={() => updateTaskStatus(task._id, 'in-progress')}>
                  Start
                </button>
              </div>
            ))}
          </div>
          
          <div className="kanban-column">
            <h3>In Progress</h3>
            {tasks.filter(t => t.status === 'in-progress').map(task => (
              <div key={task._id} className="task-card">
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <button onClick={() => updateTaskStatus(task._id, 'done')}>
                  Complete
                </button>
              </div>
            ))}
          </div>
          
          <div className="kanban-column">
            <h3>Done</h3>
            {tasks.filter(t => t.status === 'done').map(task => (
              <div key={task._id} className="task-card completed">
                <h4>{task.title}</h4>
                <p>{task.description}</p>
              </div>
            ))}
          </div>
        </div>
      </div>

      <div className="tickets-section">
        <h2>My Tickets</h2>
        <table>
          <thead>
            <tr>
              <th>Subject</th>
              <th>Category</th>
              <th>Priority</th>
              <th>Status</th>
            </tr>
          </thead>
          <tbody>
            {tickets.map(ticket => (
              <tr key={ticket._id}>
                <td>{ticket.subject}</td>
                <td>{ticket.category}</td>
                <td className={`priority-${ticket.priority}`}>{ticket.priority}</td>
                <td>{ticket.status}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Create Ticket with AI

```javascript
// frontend/src/components/CreateTicket.js
import React, { useState } from 'react';
import axios from 'axios';

const CreateTicket = () => {
  const [formData, setFormData] = useState({
    subject: '',
    description: ''
  });
  const [aiSuggestion, setAiSuggestion] = useState(null);
  const [loading, setLoading] = useState(false);

  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const getAISuggestion = async () => {
    if (!formData.subject || !formData.description) return;
    
    try {
      setLoading(true);
      const response = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/classify-ticket`,
        {
          subject: formData.subject,
          description: formData.description
        }
      );
      setAiSuggestion(response.data);
    } catch (err) {
      console.error('AI classification failed:', err);
    } finally {
      setLoading(false);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    try {
      const token = localStorage.getItem('token');
      await axios.post(
        `${process.env.REACT_APP_API_URL}/api/tickets`,
        formData,
        { headers: { 'x-auth-token': token } }
      );
      
      alert('Ticket created successfully!');
      setFormData({ subject: '', description: '' });
      setAiSuggestion(null);
    } catch (err) {
      console.error(err);
      alert('Error creating ticket');
    }
  };

  return (
    <div className="create-ticket">
      <h2>Create Support Ticket</h2>
      
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label>Subject</label>
          <input
            type="text"
            name="subject"
            value={formData.subject}
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
        
        <button type="button" onClick={getAISuggestion} disabled={loading}>
          {loading ? 'Analyzing...' : 'Get AI Suggestion'}
        </button>
        
        {aiSuggestion && (
          <div className="ai-suggestion">
            <h3>AI Classification</h3>
            <p><strong>Category:</strong> {aiSuggestion.category}</p>
            <p><strong>Priority:</strong> {aiSuggestion.priority}</p>
            <p><strong>Confidence:</strong> {(aiSuggestion.confidence * 100).toFixed(0)}%</p>
          </div>
        )}
        
        <button type="submit">Create Ticket</button>
      </form>
    </div>
  );
};

export default CreateTicket;
```

## Common Patterns

### Protected Routes

```javascript
// frontend/src/components/PrivateRoute.js
import React from 'react';
import { Route, Redirect } from 'react-router-dom';

const PrivateRoute = ({ component: Component, ...rest }) => {
  const token = localStorage.getItem('token');
  
  return (
    <Route
      
