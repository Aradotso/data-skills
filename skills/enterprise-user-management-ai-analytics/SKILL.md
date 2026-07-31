---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create admin dashboard with user tracking"
  - "add AI-based ticket classification"
  - "build user management with anomaly detection"
  - "integrate burnout prediction analytics"
  - "setup Kanban board with time tracking"
  - "deploy user management system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines user administration, task management, and support ticketing with AI-powered analytics. The system provides intelligent insights including risk detection, anomaly detection, burnout analysis, and predictive project delays. Built with React frontend, Node.js backend, and FastAPI ML service using MongoDB for data persistence.

**Key capabilities:**
- Role-based user management with JWT authentication
- Kanban task tracking with time logging
- Support ticket system with AI classification
- Real-time anomaly and risk detection
- Burnout prediction based on workload patterns
- Predictive analytics for project delays

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB (local or cloud instance)

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
EOF

# Start backend server
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=${MONGODB_URI}
MODEL_PATH=./models
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

# Start development server
npm start
```

## Architecture Components

### 1. Backend API (Node.js/Express)

**User Management Routes:**

```javascript
// backend/routes/userRoutes.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');
const {
  getUsers,
  getUser,
  createUser,
  updateUser,
  deleteUser,
  getUserStats
} = require('../controllers/userController');

// Admin-only routes
router.route('/')
  .get(protect, authorize('admin'), getUsers)
  .post(protect, authorize('admin'), createUser);

router.route('/:id')
  .get(protect, getUser)
  .put(protect, authorize('admin'), updateUser)
  .delete(protect, authorize('admin'), deleteUser);

router.get('/:id/stats', protect, getUserStats);

module.exports = router;
```

**Task Management Controller:**

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = await Task.create({
      title,
      description,
      assignedTo,
      priority,
      status: 'todo',
      dueDate,
      createdBy: req.user.id
    });

    // Check for burnout prediction
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/api/predict/burnout`,
      { userId: assignedTo }
    );

    if (mlResponse.data.burnoutRisk > 0.7) {
      // Create alert for admin
      await createAlert({
        type: 'burnout_warning',
        userId: assignedTo,
        message: 'High burnout risk detected',
        severity: 'high'
      });
    }

    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { status, timeSpent } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { 
        status,
        $push: { 
          statusHistory: {
            status,
            timestamp: Date.now(),
            timeSpent
          }
        }
      },
      { new: true, runValidators: true }
    );

    if (!task) {
      return res.status(404).json({ success: false, error: 'Task not found' });
    }

    res.status(200).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};
```

**Authentication Middleware:**

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization?.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ 
      success: false, 
      error: 'Not authorized to access this route' 
    });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id);
    next();
  } catch (error) {
    return res.status(401).json({ 
      success: false, 
      error: 'Not authorized to access this route' 
    });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        error: `User role ${req.user.role} is not authorized to access this route`
      });
    }
    next();
  };
};
```

### 2. ML Service (FastAPI)

**Main ML API:**

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from datetime import datetime, timedelta
from river import linear_model, preprocessing

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models
anomaly_detector = None
risk_predictor = None

class UserActivity(BaseModel):
    userId: str
    loginTime: str
    logoutTime: Optional[str]
    tasksCompleted: int
    timeSpent: int
    location: str
    deviceType: str

class TicketData(BaseModel):
    title: str
    description: str
    priority: Optional[str]
    category: Optional[str]

class BurnoutRequest(BaseModel):
    userId: str

@app.on_event("startup")
async def load_models():
    global anomaly_detector, risk_predictor
    try:
        anomaly_detector = joblib.load('./models/anomaly_detector.pkl')
        risk_predictor = joblib.load('./models/risk_predictor.pkl')
    except FileNotFoundError:
        # Initialize new models if not found
        anomaly_detector = preprocessing.StandardScaler() | linear_model.LinearRegression()
        risk_predictor = preprocessing.StandardScaler() | linear_model.LogisticRegression()

@app.post("/api/detect/anomaly")
async def detect_anomaly(activity: UserActivity):
    """Detect anomalous user behavior"""
    try:
        # Extract features
        features = extract_activity_features(activity)
        
        # Predict anomaly score
        score = anomaly_detector.predict_one(features)
        
        is_anomaly = score > 0.75
        
        return {
            "isAnomaly": is_anomaly,
            "score": float(score),
            "userId": activity.userId,
            "timestamp": datetime.now().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/predict/burnout")
async def predict_burnout(request: BurnoutRequest):
    """Predict burnout risk for a user"""
    try:
        from pymongo import MongoClient
        import os
        
        client = MongoClient(os.getenv('MONGODB_URI'))
        db = client['enterprise_user_mgmt']
        
        # Get user's recent activity
        tasks = list(db.tasks.find({'assignedTo': request.userId}))
        
        if not tasks:
            return {"burnoutRisk": 0.0, "factors": []}
        
        # Calculate burnout indicators
        total_tasks = len(tasks)
        overdue_tasks = sum(1 for t in tasks if t.get('dueDate') and 
                           datetime.fromisoformat(t['dueDate']) < datetime.now())
        avg_time_spent = np.mean([t.get('timeSpent', 0) for t in tasks])
        
        # Simple burnout risk calculation
        burnout_risk = min(1.0, (overdue_tasks / max(total_tasks, 1) * 0.5 + 
                                 avg_time_spent / (8 * 3600) * 0.5))
        
        factors = []
        if overdue_tasks > 5:
            factors.append("High number of overdue tasks")
        if avg_time_spent > 10 * 3600:
            factors.append("Extended working hours")
        if total_tasks > 20:
            factors.append("Heavy workload")
        
        return {
            "userId": request.userId,
            "burnoutRisk": float(burnout_risk),
            "factors": factors,
            "metrics": {
                "totalTasks": total_tasks,
                "overdueTasks": overdue_tasks,
                "avgTimeSpent": float(avg_time_spent)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/classify/ticket")
async def classify_ticket(ticket: TicketData):
    """Classify support ticket using AI"""
    try:
        # Simple keyword-based classification
        text = f"{ticket.title} {ticket.description}".lower()
        
        categories = {
            "technical": ["error", "bug", "crash", "issue", "not working"],
            "access": ["login", "password", "access", "permission", "locked"],
            "feature": ["feature", "request", "enhancement", "add", "new"],
            "performance": ["slow", "lag", "performance", "timeout", "loading"]
        }
        
        category_scores = {}
        for cat, keywords in categories.items():
            score = sum(1 for kw in keywords if kw in text)
            category_scores[cat] = score
        
        predicted_category = max(category_scores, key=category_scores.get)
        
        # Predict priority
        urgent_keywords = ["urgent", "critical", "emergency", "immediately", "asap"]
        high_keywords = ["important", "high", "soon", "needed"]
        
        if any(kw in text for kw in urgent_keywords):
            predicted_priority = "urgent"
        elif any(kw in text for kw in high_keywords):
            predicted_priority = "high"
        else:
            predicted_priority = "medium"
        
        return {
            "category": predicted_category,
            "priority": predicted_priority,
            "confidence": max(category_scores.values()) / max(sum(category_scores.values()), 1)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/predict/project-delay")
async def predict_project_delay(project_data: dict):
    """Predict if project will be delayed"""
    try:
        # Extract features
        tasks_total = project_data.get('totalTasks', 0)
        tasks_completed = project_data.get('completedTasks', 0)
        days_remaining = project_data.get('daysRemaining', 1)
        
        completion_rate = tasks_completed / max(tasks_total, 1)
        tasks_per_day_needed = (tasks_total - tasks_completed) / max(days_remaining, 1)
        historical_rate = project_data.get('historicalTasksPerDay', 1)
        
        delay_probability = max(0, min(1, tasks_per_day_needed / max(historical_rate, 0.1) - 1))
        
        estimated_delay_days = max(0, (tasks_total - tasks_completed) / max(historical_rate, 1) - days_remaining)
        
        return {
            "delayProbability": float(delay_probability),
            "estimatedDelayDays": int(estimated_delay_days),
            "recommendation": "Increase resources" if delay_probability > 0.7 else "On track"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def extract_activity_features(activity: UserActivity) -> dict:
    """Extract numerical features from activity"""
    login_hour = datetime.fromisoformat(activity.loginTime).hour
    
    return {
        'login_hour': login_hour,
        'tasks_completed': activity.tasksCompleted,
        'time_spent': activity.timeSpent,
        'is_unusual_hour': 1 if login_hour < 6 or login_hour > 22 else 0
    }
```

### 3. Frontend Components

**Admin Dashboard:**

```javascript
// frontend/src/components/AdminDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [stats, setStats] = useState({});
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    fetchDashboardData();
    const interval = setInterval(fetchDashboardData, 30000);
    return () => clearInterval(interval);
  }, []);

  const fetchDashboardData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      const [usersRes, statsRes, alertsRes] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/api/users`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/api/stats`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/api/alerts`, config)
      ]);

      setUsers(usersRes.data.data);
      setStats(statsRes.data.data);
      setAlerts(alertsRes.data.data);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
    }
  };

  const checkBurnout = async (userId) => {
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/predict/burnout`,
        { userId }
      );

      if (response.data.burnoutRisk > 0.7) {
        alert(`Warning: High burnout risk detected for this user!`);
      }
    } catch (error) {
      console.error('Error checking burnout:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p>{stats.totalUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p>{stats.activeTasks}</p>
        </div>
        <div className="stat-card">
          <h3>Pending Tickets</h3>
          <p>{stats.pendingTickets}</p>
        </div>
      </div>

      <div className="alerts-section">
        <h2>AI Alerts</h2>
        {alerts.map(alert => (
          <div key={alert._id} className={`alert alert-${alert.severity}`}>
            <strong>{alert.type}</strong>: {alert.message}
          </div>
        ))}
      </div>

      <div className="users-table">
        <h2>Users</h2>
        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Tasks</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.name}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.taskCount}</td>
                <td>
                  <button onClick={() => checkBurnout(user._id)}>
                    Check Burnout
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AdminDashboard;
```

**Kanban Board:**

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { DragDropContext, Droppable, Draggable } from 'react-beautiful-dnd';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [activeTimer, setActiveTimer] = useState(null);
  const [timeSpent, setTimeSpent] = useState(0);

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  useEffect(() => {
    let interval;
    if (activeTimer) {
      interval = setInterval(() => {
        setTimeSpent(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [activeTimer]);

  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks?assignedTo=${userId}`,
        { headers: { Authorization: `Bearer ${token}` }}
      );

      const grouped = {
        todo: response.data.data.filter(t => t.status === 'todo'),
        inProgress: response.data.data.filter(t => t.status === 'inProgress'),
        done: response.data.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const onDragEnd = async (result) => {
    if (!result.destination) return;

    const { source, destination, draggableId } = result;
    
    if (source.droppableId === destination.droppableId) return;

    try {
      const token = localStorage.getItem('token');
      await axios.put(
        `${process.env.REACT_APP_API_URL}/api/tasks/${draggableId}`,
        { 
          status: destination.droppableId,
          timeSpent: activeTimer === draggableId ? timeSpent : 0
        },
        { headers: { Authorization: `Bearer ${token}` }}
      );

      if (activeTimer === draggableId) {
        setActiveTimer(null);
        setTimeSpent(0);
      }

      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const startTimer = (taskId) => {
    setActiveTimer(taskId);
    setTimeSpent(0);
  };

  const stopTimer = () => {
    setActiveTimer(null);
    setTimeSpent(0);
  };

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <DragDropContext onDragEnd={onDragEnd}>
      <div className="kanban-board">
        {['todo', 'inProgress', 'done'].map(column => (
          <Droppable key={column} droppableId={column}>
            {(provided) => (
              <div 
                className="kanban-column"
                ref={provided.innerRef}
                {...provided.droppableProps}
              >
                <h3>{column.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
                {tasks[column].map((task, index) => (
                  <Draggable 
                    key={task._id} 
                    draggableId={task._id} 
                    index={index}
                  >
                    {(provided) => (
                      <div
                        className="kanban-card"
                        ref={provided.innerRef}
                        {...provided.draggableProps}
                        {...provided.dragHandleProps}
                      >
                        <h4>{task.title}</h4>
                        <p>{task.description}</p>
                        <div className="task-meta">
                          <span className={`priority ${task.priority}`}>
                            {task.priority}
                          </span>
                          {activeTimer === task._id && (
                            <div className="timer">
                              {formatTime(timeSpent)}
                              <button onClick={stopTimer}>Stop</button>
                            </div>
                          )}
                          {!activeTimer && column === 'inProgress' && (
                            <button onClick={() => startTimer(task._id)}>
                              Start Timer
                            </button>
                          )}
                        </div>
                      </div>
                    )}
                  </Draggable>
                ))}
                {provided.placeholder}
              </div>
            )}
          </Droppable>
        ))}
      </div>
    </DragDropContext>
  );
};

export default KanbanBoard;
```

**Ticket Creation with AI:**

```javascript
// frontend/src/components/CreateTicket.js
import React, { useState } from 'react';
import axios from 'axios';

const CreateTicket = () => {
  const [formData, setFormData] = useState({
    title: '',
    description: '',
    priority: '',
    category: ''
  });
  const [aiSuggestions, setAiSuggestions] = useState(null);
  const [isAnalyzing, setIsAnalyzing] = useState(false);

  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const analyzeTicket = async () => {
    if (!formData.title || !formData.description) {
      alert('Please fill in title and description first');
      return;
    }

    setIsAnalyzing(true);
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/classify/ticket`,
        {
          title: formData.title,
          description: formData.description
        }
      );

      setAiSuggestions(response.data);
      setFormData({
        ...formData,
        priority: response.data.priority,
        category: response.data.category
      });
    } catch (error) {
      console.error('Error analyzing ticket:', error);
    } finally {
      setIsAnalyzing(false);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    try {
      const token = localStorage.getItem('token');
      await axios.post(
        `${process.env.REACT_APP_API_URL}/api/tickets`,
        formData,
        { headers: { Authorization: `Bearer ${token}` }}
      );

      alert('Ticket created successfully!');
      setFormData({ title: '', description: '', priority: '', category: '' });
      setAiSuggestions(null);
    } catch (error) {
      console.error('Error creating ticket:', error);
      alert('Failed to create ticket');
    }
  };

  return (
    <div className="create-ticket">
      <h2>Create Support Ticket</h2>
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

        <button 
          type="button" 
          onClick={analyzeTicket}
          disabled={isAnalyzing}
        >
          {isAnalyzing ? 'Analyzing...' : 'AI Analyze'}
        </button>

        {aiSuggestions && (
          <div className="ai-suggestions">
            <h4>AI Suggestions</h4>
            <p>Category: <strong>{aiSuggestions.category}</strong></p>
            <p>Priority: <strong>{aiSuggestions.priority}</strong></p>
            <p>Confidence: {(aiSuggestions.confidence * 100).toFixed(1)}%</p>
          </div>
        )}

        <div className="form-group">
          <label>Priority</label>
          <select
            name="priority"
            value={formData.priority}
            onChange={handleChange}
            required
          >
            <option value="">Select Priority</option>
            <option value="low">Low</option>
            <option value="medium">Medium</option>
            <option value="high">High</option>
            <option value="urgent">Urgent</option>
          </select>
        </div>

        <div className="form-group">
          <label>Category</label>
          <select
            name="category"
            value={formData.category}
            onChange={handleChange}
            required
          >
            <option value="">Select Category</option>
            <option value="technical">Technical</option>
            <option value="access">Access</option>
            <option value="feature">Feature Request</option>
            <option value="performance">Performance</option>
          </select>
        </div>

        <button type="submit">Create Ticket</button>
      </form>
    </div>
  );
};

export default CreateTicket;
```

## Database Models

**User Model:**

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please add a name']
  },
  email: {
    type: String,
    required: [true, 'Please add an email'],
    unique: true,
    match: [/^\S+@\S+\.\S+$/, 'Please add a valid email']
  },
  password: {
    type: String,
    required: [true, 'Please add a password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  position: String,
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: Date,
  isActive: {
    type: Boolean,
    default: true
  }
});

UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) {
    next();
  }
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

UserSchema.methods.getSignedJwtToken = function() {
  return jwt.sign({ id: this._id }, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_EXPIRE
  });
};

UserSchema.methods.matchPassword = async function(enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

**Task Model:**

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: [true, 'Please add a task title']
  },
  description: String,
  assignedTo: {
    type: mongoose.Schema.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.ObjectId,
    ref: 'User',
    required: true
  },
  status: {
    type: String,
    enum: ['todo', 'inProgress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  dueDate: Date,
  timeSpent: {
    type: Number,
    default: 0
  },
  statusHistory: [{
    status: String,
    timestamp: Date,
    timeSpent: Number
  }],
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', TaskSchema);
```

## Configuration

### Environment
