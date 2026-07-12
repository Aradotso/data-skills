---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics for user behavior prediction
  - implement task tracking with kanban board
  - create a ticket management system with AI classification
  - build user dashboard with performance insights
  - set up JWT authentication for user management
  - deploy enterprise management system with ML service
  - configure AI-powered risk detection and anomaly analysis
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user management, task tracking, and support ticket handling with AI-powered analytics. The system provides risk detection, anomaly detection, burnout analysis, and predictive project insights using machine learning models.

**Key capabilities:**
- User authentication with JWT
- Task management with Kanban board
- AI-based ticket classification and routing
- Predictive analytics for risk and burnout detection
- Real-time performance monitoring
- Role-based access control (Admin/User)

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+ (for ML service)
- MongoDB running locally or cloud instance

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

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend server:

```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

Access the application at `http://localhost:3000`

## Architecture

The system consists of three main services:

1. **Frontend (React)**: User interface for admins and users
2. **Backend (Node.js)**: REST API handling business logic and data
3. **ML Service (FastAPI)**: AI/ML models for analytics and predictions

## Core Features Implementation

### 1. User Authentication

#### Backend - User Registration (JavaScript)

```javascript
// backend/routes/auth.js
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const router = express.Router();

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    // Check if user exists
    let user = await User.findOne({ email });
    if (user) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    // Create user
    user = new User({
      name,
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
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ message: 'Invalid credentials' });
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
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

#### Middleware - Auth Protection

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const auth = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ message: 'No token, authorization denied' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Token is not valid' });
  }
};

const adminAuth = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { auth, adminAuth };
```

### 2. Task Management with Kanban Board

#### Backend - Task Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { auth } = require('../middleware/auth');

const router = express.Router();

// Get all tasks for a user
router.get('/', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Create new task
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, priority, deadline, assignedTo } = req.body;
    
    const task = new Task({
      title,
      description,
      priority: priority || 'medium',
      status: 'todo',
      assignedTo: assignedTo || req.user.userId,
      createdBy: req.user.userId,
      deadline: deadline ? new Date(deadline) : null
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update task status (Kanban board drag & drop)
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    
    if (!['todo', 'inprogress', 'done'].includes(status)) {
      return res.status(400).json({ message: 'Invalid status' });
    }
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Track time on task
router.post('/:id/time', auth, async (req, res) => {
  try {
    const { duration } = req.body; // duration in seconds
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.timeTracked = (task.timeTracked || 0) + duration;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

#### Frontend - Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetchTasks();
  }, []);
  
  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inprogress: response.data.filter(t => t.status === 'inprogress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(categorized);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };
  
  const handleDrop = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      fetchTasks(); // Refresh tasks
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };
  
  const TaskCard = ({ task }) => (
    <div
      className="task-card"
      draggable
      onDragStart={(e) => e.dataTransfer.setData('taskId', task._id)}
    >
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority ${task.priority}`}>{task.priority}</span>
      {task.deadline && <span>Due: {new Date(task.deadline).toLocaleDateString()}</span>}
    </div>
  );
  
  const Column = ({ title, status, tasks }) => (
    <div
      className="kanban-column"
      onDragOver={(e) => e.preventDefault()}
      onDrop={(e) => {
        const taskId = e.dataTransfer.getData('taskId');
        handleDrop(taskId, status);
      }}
    >
      <h3>{title} ({tasks.length})</h3>
      <div className="task-list">
        {tasks.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
  
  if (loading) return <div>Loading...</div>;
  
  return (
    <div className="kanban-board">
      <Column title="To Do" status="todo" tasks={tasks.todo} />
      <Column title="In Progress" status="inprogress" tasks={tasks.inprogress} />
      <Column title="Done" status="done" tasks={tasks.done} />
    </div>
  );
};

export default KanbanBoard;
```

### 3. Support Ticket System with AI Classification

#### Backend - Ticket Routes

```javascript
// backend/routes/tickets.js
const express = require('express');
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { auth } = require('../middleware/auth');

const router = express.Router();

// Create new ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, category } = req.body;
    
    // Call ML service for AI classification
    let aiCategory = category;
    let aiPriority = 'medium';
    
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { title, description }
      );
      
      aiCategory = mlResponse.data.category;
      aiPriority = mlResponse.data.priority;
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      category: aiCategory,
      priority: aiPriority,
      status: 'open',
      createdBy: req.user.userId
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Get user's tickets
router.get('/', auth, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.userId })
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update ticket status (Admin)
router.patch('/:id', auth, async (req, res) => {
  try {
    const { status, response } = req.body;
    
    const ticket = await Ticket.findById(req.params.id);
    if (!ticket) {
      return res.status(404).json({ message: 'Ticket not found' });
    }
    
    if (status) ticket.status = status;
    if (response) ticket.response = response;
    
    if (status === 'resolved') {
      ticket.resolvedAt = new Date();
    }
    
    await ticket.save();
    res.json(ticket);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### 4. AI Analytics Service (Python/FastAPI)

#### ML Service - Ticket Classification

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib
import os
from typing import Optional

app = FastAPI()

# Models storage
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketInput(BaseModel):
    title: str
    description: str

class UserBehavior(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    tasksCompleted: int
    averageTaskTime: float
    ticketsRaised: int
    loginLocations: list

# Initialize or load models
try:
    ticket_vectorizer = joblib.load(f'{MODEL_PATH}/ticket_vectorizer.pkl')
    ticket_classifier = joblib.load(f'{MODEL_PATH}/ticket_classifier.pkl')
except:
    ticket_vectorizer = TfidfVectorizer(max_features=500)
    ticket_classifier = MultinomialNB()

@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketInput):
    """Classify ticket category and priority using ML"""
    try:
        # Combine title and description
        text = f"{ticket.title} {ticket.description}"
        
        # Simple rule-based classification (can be replaced with trained model)
        keywords = {
            'technical': ['bug', 'error', 'crash', 'broken', 'not working'],
            'account': ['login', 'password', 'access', 'permission'],
            'feature': ['request', 'new', 'add', 'implement'],
            'billing': ['payment', 'invoice', 'charge', 'subscription']
        }
        
        category = 'general'
        priority = 'medium'
        
        text_lower = text.lower()
        for cat, words in keywords.items():
            if any(word in text_lower for word in words):
                category = cat
                break
        
        # Priority based on urgency keywords
        urgent_words = ['urgent', 'critical', 'asap', 'immediately', 'emergency']
        if any(word in text_lower for word in urgent_words):
            priority = 'high'
        
        return {
            'category': category,
            'priority': priority,
            'confidence': 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(behavior: UserBehavior):
    """Detect anomalous user behavior for security"""
    try:
        risk_score = 0.0
        reasons = []
        
        # Check failed login attempts
        if behavior.failedLogins > 5:
            risk_score += 0.3
            reasons.append('High number of failed login attempts')
        
        # Check unusual login locations
        if len(behavior.loginLocations) > 3:
            risk_score += 0.2
            reasons.append('Multiple login locations detected')
        
        # Check task completion rate
        if behavior.tasksCompleted == 0 and behavior.ticketsRaised > 10:
            risk_score += 0.15
            reasons.append('Low task completion with high ticket activity')
        
        # Check excessive login attempts
        if behavior.loginAttempts > 50:
            risk_score += 0.25
            reasons.append('Excessive login activity')
        
        is_anomaly = risk_score > 0.5
        
        return {
            'isAnomaly': is_anomaly,
            'riskScore': min(risk_score, 1.0),
            'reasons': reasons,
            'recommendation': 'Review account activity' if is_anomaly else 'Normal behavior'
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-burnout")
async def predict_burnout(user_data: dict):
    """Predict user burnout based on workload analysis"""
    try:
        workload_hours = user_data.get('totalHours', 0)
        tasks_assigned = user_data.get('tasksAssigned', 0)
        tasks_completed = user_data.get('tasksCompleted', 0)
        overtime_hours = user_data.get('overtimeHours', 0)
        
        burnout_score = 0.0
        factors = []
        
        # High workload
        if workload_hours > 50:
            burnout_score += 0.3
            factors.append('Excessive working hours')
        
        # Low completion rate
        completion_rate = tasks_completed / tasks_assigned if tasks_assigned > 0 else 0
        if completion_rate < 0.5:
            burnout_score += 0.25
            factors.append('Low task completion rate')
        
        # Overtime
        if overtime_hours > 10:
            burnout_score += 0.25
            factors.append('High overtime hours')
        
        # High task load
        if tasks_assigned > 20:
            burnout_score += 0.2
            factors.append('High number of assigned tasks')
        
        risk_level = 'high' if burnout_score > 0.7 else 'medium' if burnout_score > 0.4 else 'low'
        
        return {
            'burnoutScore': min(burnout_score, 1.0),
            'riskLevel': risk_level,
            'factors': factors,
            'recommendation': 'Reduce workload and redistribute tasks' if risk_level == 'high' else 'Monitor workload'
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-project-delay")
async def predict_project_delay(project_data: dict):
    """Predict if project will be delayed based on current metrics"""
    try:
        total_tasks = project_data.get('totalTasks', 0)
        completed_tasks = project_data.get('completedTasks', 0)
        days_remaining = project_data.get('daysRemaining', 0)
        average_completion_time = project_data.get('avgCompletionTime', 0)
        
        # Calculate remaining tasks
        remaining_tasks = total_tasks - completed_tasks
        
        # Estimate time needed
        estimated_days = remaining_tasks * average_completion_time
        
        delay_probability = 0.0
        if estimated_days > days_remaining:
            delay_probability = min((estimated_days - days_remaining) / days_remaining, 1.0)
        
        return {
            'delayProbability': delay_probability,
            'estimatedDelay': max(0, estimated_days - days_remaining),
            'isAtRisk': delay_probability > 0.5,
            'recommendation': 'Increase resources or extend deadline' if delay_probability > 0.5 else 'On track'
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

### 5. Admin Dashboard with Analytics

#### Backend - Analytics Route

```javascript
// backend/routes/analytics.js
const express = require('express');
const axios = require('axios');
const User = require('../models/User');
const Task = require('../models/Task');
const Ticket = require('../models/Ticket');
const { auth, adminAuth } = require('../middleware/auth');

const router = express.Router();

// Get organization analytics
router.get('/overview', auth, adminAuth, async (req, res) => {
  try {
    const totalUsers = await User.countDocuments();
    const totalTasks = await Task.countDocuments();
    const totalTickets = await Ticket.countDocuments();
    
    const activeTasks = await Task.countDocuments({ status: { $ne: 'done' } });
    const openTickets = await Ticket.countDocuments({ status: 'open' });
    
    // Task completion rate
    const completedTasks = await Task.countDocuments({ status: 'done' });
    const completionRate = totalTasks > 0 ? (completedTasks / totalTasks) * 100 : 0;
    
    res.json({
      totalUsers,
      totalTasks,
      totalTickets,
      activeTasks,
      openTickets,
      completionRate: completionRate.toFixed(2)
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Get user performance analytics
router.get('/user-performance/:userId', auth, adminAuth, async (req, res) => {
  try {
    const userId = req.params.userId;
    
    const tasks = await Task.find({ assignedTo: userId });
    const tickets = await Ticket.find({ createdBy: userId });
    
    const completedTasks = tasks.filter(t => t.status === 'done').length;
    const totalHours = tasks.reduce((sum, t) => sum + (t.timeTracked || 0), 0) / 3600;
    
    const avgTaskTime = completedTasks > 0 
      ? totalHours / completedTasks 
      : 0;
    
    // Call ML service for burnout prediction
    let burnoutAnalysis = null;
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/predict-burnout`,
        {
          totalHours,
          tasksAssigned: tasks.length,
          tasksCompleted: completedTasks,
          overtimeHours: Math.max(0, totalHours - 40)
        }
      );
      burnoutAnalysis = mlResponse.data;
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }
    
    res.json({
      userId,
      tasksAssigned: tasks.length,
      tasksCompleted: completedTasks,
      totalHours: totalHours.toFixed(2),
      avgTaskTime: avgTaskTime.toFixed(2),
      ticketsRaised: tickets.length,
      burnoutAnalysis
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Check for anomalies
router.post('/check-anomaly/:userId', auth, adminAuth, async (req, res) => {
  try {
    const userId = req.params.userId;
    const user = await User.findById(userId);
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    const tasks = await Task.find({ assignedTo: userId });
    const tickets = await Ticket.find({ createdBy: userId });
    
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/detect-anomaly`,
      {
        userId,
        loginAttempts: user.loginAttempts || 0,
        failedLogins: user.failedLogins || 0,
        tasksCompleted: tasks.filter(t => t.status === 'done').length,
        averageTaskTime: 2.5,
        ticketsRaised: tickets.length,
        loginLocations: user.loginLocations || []
      }
    );
    
    res.json(mlResponse.data);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
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
  loginAttempts: {
    type: Number,
    default: 0
  },
  failedLogins: {
    type: Number,
    default: 0
  },
  loginLocations: [{
    type: String
  }],
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
  description: {
    type: String,
    required: true
  },
  status: {
    type: String,
    enum: ['todo', 'inprogress', 'done'],
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
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  deadline: {
    type: Date
  },
  timeTracked: {
    type: Number, // in seconds
    default: 0
  },
  completedAt: {
    type: Date
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

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
    enum: ['technical', 'account', 'feature', 'billing', 'general'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
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
  response: {
    type: String
  },
  resolvedAt: {
    type: Date
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## API Configuration

### Backend Server Setup

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const authRoutes = require('./routes/auth');
const taskRoutes = require('./routes/tasks');
const ticketRoutes = require('./routes/tickets');
const analyticsRoutes = require('./routes/analytics');
const userRoutes = require('./routes/users');

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// MongoDB Connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
