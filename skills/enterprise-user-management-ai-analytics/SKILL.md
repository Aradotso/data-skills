---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered risk detection, anomaly analysis, and task management using React, Node.js, FastAPI, and MongoDB
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics into user management
  - implement role-based access control with JWT authentication
  - create a task management system with Kanban board
  - add AI risk prediction and anomaly detection
  - build a ticket classification system with ML
  - deploy user management with AI insights
  - configure MongoDB for enterprise user system
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a full-stack enterprise user management system that combines traditional CRUD operations with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and ticket classification. Built with React frontend, Node.js/Express backend, FastAPI ML service, and MongoDB database.

## What This Project Does

- **User Management**: Admin dashboard for managing users with role-based access control
- **Task Management**: Kanban-style task board with time tracking and progress monitoring
- **Support Tickets**: AI-powered ticket classification and routing system
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project insights
- **Authentication**: JWT-based secure authentication system
- **Audit Logging**: Track user activities and suspicious behavior

## Installation

### Prerequisites

```bash
# Required software
node >= 14.x
python >= 3.8
mongodb >= 4.4
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env  # Configure environment variables
npm start  # Runs on http://localhost:5000

# ML Service setup
cd ../ml-service
pip install -r requirements.txt
cp .env.example .env  # Configure ML service variables
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup
cd ../frontend
npm install
cp .env.example .env  # Configure API endpoints
npm start  # Runs on http://localhost:3000
```

## Environment Configuration

### Backend (.env)

```bash
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
# Or MongoDB Atlas
# MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/dbname

# JWT Authentication
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# CORS
FRONTEND_URL=http://localhost:3000
```

### ML Service (.env)

```bash
# FastAPI
HOST=0.0.0.0
PORT=8000

# Models
MODEL_PATH=./models
RISK_THRESHOLD=0.7
ANOMALY_THRESHOLD=0.8
BURNOUT_THRESHOLD=0.75
```

### Frontend (.env)

```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

## Key Backend API Endpoints

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register user
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }
    
    // Create user
    const user = new User({ name, email, password, role });
    await user.save();
    
    // Generate token
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({ token, user: user.toJSON() });
  } catch (error) {
    res.status(500).json({ error: error.message });
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
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({ token, user: user.toJSON() });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### User Management

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { authMiddleware, adminOnly } = require('../middleware/auth');

// Get all users (admin only)
router.get('/', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user by ID
router.get('/:id', authMiddleware, async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update user
router.put('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const { name, email, role, status } = req.body;
    
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { name, email, role, status },
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Delete user
router.delete('/:id', authMiddleware, adminOnly, async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      createdBy: req.user.userId,
      status: 'todo'
    });
    
    await task.save();
    await task.populate('assignedTo createdBy', 'name email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const filter = {};
    
    // Regular users see only their tasks
    if (req.user.role !== 'admin') {
      filter.assignedTo = req.user.userId;
    }
    
    const tasks = await Task.find(filter)
      .populate('assignedTo createdBy', 'name email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    ).populate('assignedTo createdBy', 'name email');
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { timeSpent } = req.body;
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    task.timeTracking.push({
      userId: req.user.userId,
      duration: timeSpent,
      date: new Date()
    });
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service API

### FastAPI ML Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
from river import anomaly
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
risk_model = None
anomaly_detector = None
burnout_model = None

@app.on_event("startup")
async def load_models():
    global risk_model, anomaly_detector, burnout_model
    
    model_path = os.getenv("MODEL_PATH", "./models")
    
    # Load or initialize models
    try:
        risk_model = joblib.load(f"{model_path}/risk_model.pkl")
    except:
        risk_model = RandomForestClassifier(n_estimators=100)
    
    try:
        anomaly_detector = joblib.load(f"{model_path}/anomaly_model.pkl")
    except:
        anomaly_detector = IsolationForest(contamination=0.1)
    
    try:
        burnout_model = joblib.load(f"{model_path}/burnout_model.pkl")
    except:
        burnout_model = RandomForestClassifier(n_estimators=100)

# Ticket Classification
class TicketInput(BaseModel):
    title: str
    description: str
    priority: str

class TicketClassification(BaseModel):
    category: str
    urgency_score: float
    recommended_assignee: Optional[str]

@app.post("/classify-ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketInput):
    try:
        # Simple keyword-based classification
        text = f"{ticket.title} {ticket.description}".lower()
        
        categories = {
            "technical": ["bug", "error", "crash", "performance", "api"],
            "account": ["login", "password", "access", "permission"],
            "feature": ["request", "enhancement", "suggest", "add"],
            "billing": ["payment", "invoice", "subscription", "charge"]
        }
        
        category = "general"
        max_matches = 0
        
        for cat, keywords in categories.items():
            matches = sum(1 for keyword in keywords if keyword in text)
            if matches > max_matches:
                max_matches = matches
                category = cat
        
        # Urgency score based on priority
        urgency_map = {"low": 0.3, "medium": 0.6, "high": 0.9, "critical": 1.0}
        urgency_score = urgency_map.get(ticket.priority.lower(), 0.5)
        
        return TicketClassification(
            category=category,
            urgency_score=urgency_score,
            recommended_assignee=None
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Risk Prediction
class UserActivity(BaseModel):
    user_id: str
    login_attempts: int
    failed_logins: int
    data_access_count: int
    unusual_hours: int
    location_changes: int

class RiskPrediction(BaseModel):
    risk_score: float
    risk_level: str
    factors: List[str]

@app.post("/predict-risk", response_model=RiskPrediction)
async def predict_risk(activity: UserActivity):
    try:
        # Calculate risk score
        risk_score = 0.0
        factors = []
        
        # Failed login attempts
        if activity.failed_logins > 3:
            risk_score += 0.3
            factors.append("Multiple failed login attempts")
        
        # Unusual hours access
        if activity.unusual_hours > 5:
            risk_score += 0.2
            factors.append("Access during unusual hours")
        
        # Location changes
        if activity.location_changes > 2:
            risk_score += 0.25
            factors.append("Multiple location changes")
        
        # Excessive data access
        if activity.data_access_count > 100:
            risk_score += 0.25
            factors.append("Excessive data access")
        
        risk_score = min(risk_score, 1.0)
        
        # Risk level
        if risk_score < 0.3:
            risk_level = "low"
        elif risk_score < 0.6:
            risk_level = "medium"
        else:
            risk_level = "high"
        
        return RiskPrediction(
            risk_score=risk_score,
            risk_level=risk_level,
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Anomaly Detection
class AnomalyInput(BaseModel):
    features: List[float]

class AnomalyOutput(BaseModel):
    is_anomaly: bool
    anomaly_score: float

@app.post("/detect-anomaly", response_model=AnomalyOutput)
async def detect_anomaly(data: AnomalyInput):
    try:
        features = np.array(data.features).reshape(1, -1)
        
        # Use Isolation Forest
        prediction = anomaly_detector.predict(features)[0]
        score = anomaly_detector.score_samples(features)[0]
        
        is_anomaly = prediction == -1
        anomaly_score = abs(score)
        
        return AnomalyOutput(
            is_anomaly=is_anomaly,
            anomaly_score=float(anomaly_score)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout Detection
class WorkloadData(BaseModel):
    hours_worked: float
    tasks_completed: int
    tasks_pending: int
    overtime_hours: float
    days_without_break: int

class BurnoutPrediction(BaseModel):
    burnout_risk: float
    recommendation: str

@app.post("/predict-burnout", response_model=BurnoutPrediction)
async def predict_burnout(workload: WorkloadData):
    try:
        burnout_risk = 0.0
        
        # Hours worked
        if workload.hours_worked > 50:
            burnout_risk += 0.3
        
        # Overtime
        if workload.overtime_hours > 10:
            burnout_risk += 0.2
        
        # Days without break
        if workload.days_without_break > 10:
            burnout_risk += 0.3
        
        # Task overload
        if workload.tasks_pending > 20:
            burnout_risk += 0.2
        
        burnout_risk = min(burnout_risk, 1.0)
        
        # Recommendation
        if burnout_risk < 0.3:
            recommendation = "Workload is healthy"
        elif burnout_risk < 0.6:
            recommendation = "Consider reducing workload"
        else:
            recommendation = "High burnout risk - immediate action needed"
        
        return BurnoutPrediction(
            burnout_risk=burnout_risk,
            recommendation=recommendation
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

## Frontend React Components

### Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    setUser(user);
    
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks`);
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(categorized);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority ${task.priority}`}>{task.priority}</span>
    </div>
  );

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <TaskCard key={task._id} task={task} />
        ))}
      </div>
      
      <div className="kanban-column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <TaskCard key={task._id} task={task} />
        ))}
      </div>
      
      <div className="kanban-column">
        <h3>Done</h3>
        {tasks.done.map(task => (
          <TaskCard key={task._id} task={task} />
        ))}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './AIAnalytics.css';

const AIAnalytics = ({ userId }) => {
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      setLoading(true);
      
      // Fetch user activity data
      const activityRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/analytics/activity/${userId}`
      );
      
      // Get risk prediction
      const riskRes = await axios.post(
        `${process.env.REACT_APP_ML_SERVICE_URL}/predict-risk`,
        activityRes.data
      );
      setRiskData(riskRes.data);
      
      // Get burnout prediction
      const workloadRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/analytics/workload/${userId}`
      );
      const burnoutRes = await axios.post(
        `${process.env.REACT_APP_ML_SERVICE_URL}/predict-burnout`,
        workloadRes.data
      );
      setBurnoutData(burnoutRes.data);
      
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="analytics-card">
        <h3>Risk Assessment</h3>
        <div className={`risk-level ${riskData.risk_level}`}>
          {riskData.risk_level.toUpperCase()}
        </div>
        <p>Risk Score: {(riskData.risk_score * 100).toFixed(1)}%</p>
        <ul>
          {riskData.factors.map((factor, idx) => (
            <li key={idx}>{factor}</li>
          ))}
        </ul>
      </div>
      
      <div className="analytics-card">
        <h3>Burnout Detection</h3>
        <div className="burnout-meter">
          <div 
            className="burnout-fill" 
            style={{ width: `${burnoutData.burnout_risk * 100}%` }}
          />
        </div>
        <p>{burnoutData.recommendation}</p>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Database Models

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
    lowercase: true,
    trim: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['admin', 'user', 'manager'],
    default: 'user'
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  department: String,
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: Date
}, {
  timestamps: true
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

// Remove password from JSON
userSchema.methods.toJSON = function() {
  const obj = this.toObject();
  delete obj.password;
  return obj;
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
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: Date,
  timeTracking: [{
    userId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User'
    },
    duration: Number, // in minutes
    date: Date
  }],
  tags: [String]
}, {
  timestamps: true
});

module.exports = mongoose.model('Task', taskSchema);
```

## Common Patterns

### Protected Route Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);
    
    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }
    
    req.user = { userId: user._id, role: user.role };
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### API Service Helper

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

const mlApi = axios.create({
  baseURL: process.env.REACT_APP_ML_SERVICE_URL
});

// Add token to requests
api.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle token expiration
api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export { api, mlApi };
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const options = {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    };
    
    await mongoose.connect(process.env.MONGODB_URI, options);
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected');
});

mongoose.connection.on('error', (err) => {
  console.error('MongoDB error:', err);
});

module.exports = connectDB;
```

### CORS Configuration

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');

const app = express();

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http
