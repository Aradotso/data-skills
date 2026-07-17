---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task tracking
triggers:
  - "build an enterprise user management system"
  - "implement AI-powered user analytics"
  - "create a task management system with AI insights"
  - "set up user management with risk detection"
  - "develop burnout detection for employees"
  - "build a support ticket system with AI classification"
  - "create admin dashboard with predictive analytics"
  - "implement JWT authentication with role-based access"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack JavaScript application that combines traditional user management with AI-powered analytics. It provides user authentication, task tracking with Kanban boards, support ticket management, and intelligent insights including risk prediction, anomaly detection, burnout analysis, and predictive project management.

The system consists of three main components:
- **Frontend**: React.js application for user and admin interfaces
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI service with scikit-learn and River for real-time ML predictions

## Installation

### Prerequisites

```bash
# Required software
node --version  # v14+ required
npm --version   # v6+ required
python --version  # v3.8+ required
```

### Clone and Setup

```bash
# Clone the repository
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
# Server runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGO_URI=${MONGODB_URI}
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
# Service runs at http://localhost:8000
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
# Application runs at http://localhost:3000
```

## Architecture

### Backend API Structure

```javascript
// backend/server.js - Main entry point
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const jwt = require('jsonwebtoken');

const app = express();

app.use(cors());
app.use(express.json());

// MongoDB connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => console.log('MongoDB connected'))
  .catch(err => console.error('MongoDB connection error:', err));

// Import routes
const authRoutes = require('./routes/auth');
const userRoutes = require('./routes/users');
const taskRoutes = require('./routes/tasks');
const ticketRoutes = require('./routes/tickets');
const analyticsRoutes = require('./routes/analytics');

app.use('/api/auth', authRoutes);
app.use('/api/users', userRoutes);
app.use('/api/tasks', taskRoutes);
app.use('/api/tickets', ticketRoutes);
app.use('/api/analytics', analyticsRoutes);

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### User Model

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
}, { timestamps: true });

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 12);
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
    default: 0 // in minutes
  },
  estimatedTime: Number,
  tags: [String],
  createdAt: {
    type: Date,
    default: Date.now
  }
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  ticketId: {
    type: String,
    unique: true,
    required: true
  },
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
    enum: ['technical', 'account', 'feature', 'bug', 'other'],
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
    suggestedPriority: String
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
}, { timestamps: true });

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Authentication & Authorization

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  try {
    let token;
    
    if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
      token = req.headers.authorization.split(' ')[1];
    }
    
    if (!token) {
      return res.status(401).json({ message: 'Not authorized to access this route' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    
    if (!req.user) {
      return res.status(401).json({ message: 'User not found' });
    }
    
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Not authorized' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: `User role ${req.user.role} is not authorized to access this route` 
      });
    }
    next();
  };
};
```

### Auth Routes

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Generate JWT
const generateToken = (id) => {
  return jwt.sign({ id }, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_EXPIRE
  });
};

// Register user
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const user = await User.create({
      name,
      email,
      password,
      role: role || 'user'
    });
    
    const token = generateToken(user._id);
    
    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    if (!email || !password) {
      return res.status(400).json({ message: 'Please provide email and password' });
    }
    
    const user = await User.findOne({ email });
    
    if (!user || !(await user.comparePassword(password))) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    user.lastLogin = new Date();
    await user.save();
    
    const token = generateToken(user._id);
    
    res.json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
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

## Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { protect, authorize } = require('../middleware/auth');

// Get all tasks for authenticated user
router.get('/', protect, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user._id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .populate('assignedBy', 'name')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, data: tasks });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create new task
router.post('/', protect, authorize('admin', 'manager'), async (req, res) => {
  try {
    const taskData = {
      ...req.body,
      assignedBy: req.user._id
    };
    
    const task = await Task.create(taskData);
    await task.populate('assignedTo', 'name email');
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    // Check authorization
    if (req.user.role !== 'admin' && task.assignedTo.toString() !== req.user._id.toString()) {
      return res.status(403).json({ message: 'Not authorized' });
    }
    
    task.status = status;
    await task.save();
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Track time on task
router.patch('/:id/track-time', protect, async (req, res) => {
  try {
    const { minutes } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.timeTracked = (task.timeTracked || 0) + minutes;
    await task.save();
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

module.exports = router;
```

## Ticket Management with AI

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { protect } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, category } = req.body;
    
    // Generate unique ticket ID
    const ticketId = `TKT-${Date.now()}-${Math.random().toString(36).substr(2, 9).toUpperCase()}`;
    
    // Call ML service for AI classification
    let aiClassification = {};
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        title,
        description,
        category
      });
      aiClassification = mlResponse.data;
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }
    
    const ticket = await Ticket.create({
      ticketId,
      title,
      description,
      category,
      priority: aiClassification.suggestedPriority || 'medium',
      createdBy: req.user._id,
      aiClassification
    });
    
    await ticket.populate('createdBy', 'name email');
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Get all tickets
router.get('/', protect, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user._id };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'name email')
      .populate('assignedTo', 'name email')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, data: tickets });
  } catch (error) {
    res.status(500).json({ message: error.message });
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
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import LabelEncoder
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models directory
MODEL_DIR = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_DIR, exist_ok=True)

# Load or initialize models
try:
    ticket_classifier = joblib.load(f'{MODEL_DIR}/ticket_classifier.pkl')
    risk_model = joblib.load(f'{MODEL_DIR}/risk_model.pkl')
except:
    ticket_classifier = RandomForestClassifier(n_estimators=100)
    risk_model = RandomForestClassifier(n_estimators=100)

class TicketInput(BaseModel):
    title: str
    description: str
    category: Optional[str] = None

class UserActivityInput(BaseModel):
    userId: str
    loginFrequency: float
    taskCompletionRate: float
    averageTaskTime: float
    failedLogins: int
    afterHoursActivity: float

class BurnoutInput(BaseModel):
    userId: str
    tasksAssigned: int
    tasksCompleted: int
    averageWorkHours: float
    overtimeHours: float
    missedDeadlines: int

@app.get("/")
def read_root():
    return {"message": "Enterprise User Management ML Service", "status": "active"}

@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketInput):
    """Classify support ticket and suggest priority"""
    try:
        # Simple rule-based classification
        keywords_high = ['urgent', 'critical', 'down', 'error', 'broken', 'security']
        keywords_medium = ['issue', 'problem', 'help', 'question']
        
        text = f"{ticket.title} {ticket.description}".lower()
        
        # Determine priority
        if any(keyword in text for keyword in keywords_high):
            suggested_priority = 'high'
            confidence = 0.85
        elif any(keyword in text for keyword in keywords_medium):
            suggested_priority = 'medium'
            confidence = 0.70
        else:
            suggested_priority = 'low'
            confidence = 0.60
        
        # Classify category if not provided
        if not ticket.category:
            if any(word in text for word in ['bug', 'error', 'crash']):
                category = 'bug'
            elif any(word in text for word in ['feature', 'enhancement']):
                category = 'feature'
            elif any(word in text for word in ['account', 'login', 'password']):
                category = 'account'
            else:
                category = 'technical'
        else:
            category = ticket.category
        
        return {
            "category": category,
            "confidence": confidence,
            "suggestedPriority": suggested_priority
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-risk")
async def detect_risk(activity: UserActivityInput):
    """Detect risk based on user activity patterns"""
    try:
        risk_score = 0
        risk_factors = []
        
        # Low login frequency
        if activity.loginFrequency < 0.3:
            risk_score += 20
            risk_factors.append("Low login frequency")
        
        # Low task completion rate
        if activity.taskCompletionRate < 0.5:
            risk_score += 25
            risk_factors.append("Low task completion rate")
        
        # High failed logins
        if activity.failedLogins > 3:
            risk_score += 30
            risk_factors.append("Multiple failed login attempts")
        
        # Unusual after-hours activity
        if activity.afterHoursActivity > 0.7:
            risk_score += 15
            risk_factors.append("High after-hours activity")
        
        # Very slow task completion
        if activity.averageTaskTime > 10:
            risk_score += 10
            risk_factors.append("Slow task completion")
        
        # Determine risk level
        if risk_score >= 60:
            risk_level = "high"
        elif risk_score >= 30:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return {
            "userId": activity.userId,
            "riskScore": min(risk_score, 100),
            "riskLevel": risk_level,
            "riskFactors": risk_factors,
            "recommendation": get_risk_recommendation(risk_level, risk_factors)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(burnout: BurnoutInput):
    """Detect employee burnout risk"""
    try:
        burnout_score = 0
        indicators = []
        
        # High workload
        if burnout.tasksAssigned > 20:
            burnout_score += 25
            indicators.append("High task volume")
        
        # Low completion rate
        completion_rate = burnout.tasksCompleted / max(burnout.tasksAssigned, 1)
        if completion_rate < 0.6:
            burnout_score += 20
            indicators.append("Low completion rate")
        
        # Long work hours
        if burnout.averageWorkHours > 50:
            burnout_score += 30
            indicators.append("Excessive work hours")
        
        # High overtime
        if burnout.overtimeHours > 15:
            burnout_score += 15
            indicators.append("High overtime")
        
        # Missed deadlines
        if burnout.missedDeadlines > 3:
            burnout_score += 10
            indicators.append("Multiple missed deadlines")
        
        # Determine burnout risk
        if burnout_score >= 60:
            risk_level = "critical"
        elif burnout_score >= 40:
            risk_level = "high"
        elif burnout_score >= 20:
            risk_level = "moderate"
        else:
            risk_level = "low"
        
        return {
            "userId": burnout.userId,
            "burnoutScore": min(burnout_score, 100),
            "riskLevel": risk_level,
            "indicators": indicators,
            "recommendations": get_burnout_recommendations(risk_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_risk_recommendation(risk_level, risk_factors):
    if risk_level == "high":
        return "Immediate review required. Consider account suspension and security audit."
    elif risk_level == "medium":
        return "Monitor closely. Review recent activities and contact user if necessary."
    else:
        return "No immediate action required. Continue routine monitoring."

def get_burnout_recommendations(risk_level):
    recommendations = {
        "critical": [
            "Immediate intervention required",
            "Redistribute workload urgently",
            "Schedule mandatory time off",
            "One-on-one meeting with manager"
        ],
        "high": [
            "Reduce task assignments",
            "Offer flexible work hours",
            "Provide additional support",
            "Check in weekly"
        ],
        "moderate": [
            "Monitor workload closely",
            "Offer wellness resources",
            "Regular check-ins"
        ],
        "low": [
            "Continue current support",
            "Maintain open communication"
        ]
    }
    return recommendations.get(risk_level, [])

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Requirements for ML Service

```python
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
scikit-learn==1.3.2
numpy==1.24.3
joblib==1.3.2
python-dotenv==1.0.0
river==0.19.0
pandas==2.1.3
```

## Frontend Implementation

### React Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(response.data.user);
    } catch (error) {
      console.error('Failed to fetch user:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    try {
      const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
        email,
        password
      });
      const { token, user } = response.data;
      
      localStorage.setItem('token', token);
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      setToken(token);
      setUser(user);
      
      return { success: true };
    } catch (error) {
      return { 
        success: false, 
        message: error.response?.data?.message || 'Login failed' 
      };
    }
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setToken(null);
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading, isAuthenticated: !!user }}>
      {children}
    </AuthContext.Provider>
  );
};

export default AuthContext;
```

### Task Dashboard Component

```javascript
// frontend/src/components/TaskDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './TaskDashboard.css';

const TaskDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks`);
      const tasksByStatus = {
        todo: [],
        inprogress: [],
        done: []
      };
      
      response.data.data.forEach(task => {
        tasksByStatus[task.status].push(task);
      });
      
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        <span className="time-tracked">{task.timeTracked || 0} min</span>
      </div>
      <div className="task-actions">
        {task.status !== 'done' && (
          <button onClick={() => updateTaskStatus(task._id, 'done')}>
            Mark Done
          </button>
        )}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="task-dashboard">
      <h2>Task Board</h2>
      <div className="kanban-board">
        <div className="kanban-column">
          <h3>To Do ({tasks.todo.length})</h3>
          <div className="task-list">
            {tasks.todo.map(task => (
              <TaskCard key={task._id} task={task} />
            ))}
          </div>
        </div>
        
        <div className="kanban-column">
          <h3>In Progress ({tasks.inprogress.length})</h3>
          <div className="task-list">
            {tasks.inprogress.map(task => (
              <TaskCard key={task._id} task={task} />
            ))}
          </div>
        </div>
        
        <div className="kanban-column">
          <h3>Done ({tasks.done.length})</h3>
          <div className="task-list">
            {tasks.done.map(task => (
              <TaskCard key={task._id} task={task} />
            ))}
          </div>
        </div>
      </div>
    </div>
  );
};

export default TaskDashboard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './AIAnalytics.css';

const AIAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    burnoutUsers: [],
