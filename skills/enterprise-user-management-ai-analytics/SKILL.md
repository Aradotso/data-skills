---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, risk detection, and burnout analysis
triggers:
  - "build a user management system with AI analytics"
  - "implement task tracking with burnout detection"
  - "create an admin dashboard for user management"
  - "set up AI-powered ticket classification"
  - "add anomaly detection to user management"
  - "build a kanban board with time tracking"
  - "implement JWT authentication for enterprise app"
  - "create AI risk prediction for users"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack JavaScript application that provides comprehensive user, task, and support ticket management with integrated AI capabilities. The system features role-based access control, Kanban-style task boards, time tracking, and AI-powered insights including risk detection, anomaly detection, burnout analysis, and predictive analytics.

**Key Components:**
- **Frontend**: React.js application for user and admin interfaces
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based AI service using scikit-learn and River
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip (for ML service)
python --version
pip --version

# MongoDB running locally or connection string
```

### Clone and Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../frontend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt
```

### Environment Configuration

**Backend (.env in backend/):**
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
```

**Frontend (.env in frontend/):**
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

**ML Service (.env in ml-service/):**
```env
MODEL_PATH=./models
DB_CONNECTION=mongodb://localhost:27017/enterprise_user_mgmt
```

## Running the Application

### Start Backend Server

```bash
cd backend
npm start
# Backend runs at http://localhost:5000
```

### Start ML Service

```bash
cd ml-service
uvicorn main:app --reload
# ML service runs at http://localhost:8000
```

### Start Frontend

```bash
cd frontend
npm start
# Frontend runs at http://localhost:3000
```

## Backend API Structure

### Authentication

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
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate JWT token
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ success: false, error: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ success: false, error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({
      success: true,
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

### Middleware for Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

// Protect routes - require authentication
exports.protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ success: false, error: 'Not authorized' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ success: false, error: 'Token invalid' });
  }
};

// Require admin role
exports.adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ success: false, error: 'Admin access required' });
  }
  next();
};
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { protect } = require('../middleware/auth');

// Get all tasks for logged-in user
router.get('/', protect, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, data: tasks });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Create new task
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, priority, assignedTo, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      priority,
      assignedTo,
      assignedBy: req.user.id,
      dueDate,
      status: 'todo'
    });
    
    await task.save();
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Update task status
router.put('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body; // todo, in-progress, done
    
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ success: false, error: 'Task not found' });
    }
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Track time on task
router.post('/:id/time', protect, async (req, res) => {
  try {
    const { timeSpent } = req.body; // time in minutes
    
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { $inc: { timeTracked: timeSpent } },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ success: false, error: 'Task not found' });
    }
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

### Support Tickets

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const { protect, adminOnly } = require('../middleware/auth');
const axios = require('axios');

// Create support ticket
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, category } = req.body;
    
    const ticket = new Ticket({
      title,
      description,
      category,
      createdBy: req.user.id,
      status: 'open'
    });
    
    // Call ML service for ticket classification
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { title, description }
      );
      
      ticket.priority = mlResponse.data.priority;
      ticket.suggestedCategory = mlResponse.data.category;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    await ticket.save();
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Get tickets (admin sees all, users see their own)
router.get('/', protect, async (req, res) => {
  try {
    const query = req.user.role === 'admin' ? {} : { createdBy: req.user.id };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, data: tickets });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Assign ticket (admin only)
router.put('/:id/assign', protect, adminOnly, async (req, res) => {
  try {
    const { assignedTo } = req.body;
    
    const ticket = await Ticket.findByIdAndUpdate(
      req.params.id,
      { assignedTo, status: 'in-progress' },
      { new: true }
    );
    
    if (!ticket) {
      return res.status(404).json({ success: false, error: 'Ticket not found' });
    }
    
    res.json({ success: true, data: ticket });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

## ML Service API

### FastAPI Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List
import joblib
import numpy as np
from datetime import datetime

app = FastAPI(title="Enterprise User Management ML Service")

# Load pre-trained models (train these first)
try:
    ticket_classifier = joblib.load('models/ticket_classifier.pkl')
    risk_predictor = joblib.load('models/risk_predictor.pkl')
    burnout_detector = joblib.load('models/burnout_detector.pkl')
except FileNotFoundError:
    print("Warning: Models not found. Please train models first.")

class TicketRequest(BaseModel):
    title: str
    description: str

class TicketResponse(BaseModel):
    priority: str
    category: str
    confidence: float

class UserMetrics(BaseModel):
    user_id: str
    tasks_completed: int
    tasks_pending: int
    avg_task_time: float
    tickets_raised: int
    login_frequency: float
    working_hours_per_day: float

class RiskResponse(BaseModel):
    risk_level: str
    risk_score: float
    factors: List[str]

@app.post("/classify-ticket", response_model=TicketResponse)
async def classify_ticket(ticket: TicketRequest):
    """
    Classify ticket priority and category using NLP
    """
    try:
        # Combine title and description for classification
        text = f"{ticket.title} {ticket.description}"
        
        # Feature extraction (TF-IDF or similar)
        # This is simplified - in production use proper text vectorization
        features = extract_text_features(text)
        
        # Predict priority and category
        priority_pred = ticket_classifier.predict_priority(features)
        category_pred = ticket_classifier.predict_category(features)
        confidence = ticket_classifier.predict_proba(features).max()
        
        return TicketResponse(
            priority=priority_pred[0],
            category=category_pred[0],
            confidence=float(confidence)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk", response_model=RiskResponse)
async def predict_risk(metrics: UserMetrics):
    """
    Predict user risk based on behavioral patterns
    """
    try:
        # Create feature vector
        features = np.array([[
            metrics.tasks_completed,
            metrics.tasks_pending,
            metrics.avg_task_time,
            metrics.tickets_raised,
            metrics.login_frequency,
            metrics.working_hours_per_day
        ]])
        
        # Predict risk score
        risk_score = risk_predictor.predict_proba(features)[0][1]
        
        # Determine risk level
        if risk_score > 0.7:
            risk_level = "high"
        elif risk_score > 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        # Identify risk factors
        factors = []
        if metrics.tasks_pending > 10:
            factors.append("High pending tasks")
        if metrics.avg_task_time > 480:  # 8 hours
            factors.append("Slow task completion")
        if metrics.login_frequency < 0.5:
            factors.append("Low engagement")
        
        return RiskResponse(
            risk_level=risk_level,
            risk_score=float(risk_score),
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(metrics: UserMetrics):
    """
    Detect employee burnout based on workload and behavior
    """
    try:
        features = np.array([[
            metrics.working_hours_per_day,
            metrics.tasks_pending,
            metrics.avg_task_time,
            metrics.tickets_raised,
            metrics.login_frequency
        ]])
        
        burnout_score = burnout_detector.predict_proba(features)[0][1]
        
        burnout_risk = "high" if burnout_score > 0.6 else "medium" if burnout_score > 0.3 else "low"
        
        recommendations = []
        if metrics.working_hours_per_day > 10:
            recommendations.append("Reduce working hours")
        if metrics.tasks_pending > 15:
            recommendations.append("Redistribute tasks")
        if burnout_score > 0.6:
            recommendations.append("Schedule wellness check-in")
        
        return {
            "burnout_risk": burnout_risk,
            "burnout_score": float(burnout_score),
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(metrics: UserMetrics):
    """
    Detect anomalous user behavior for security
    """
    try:
        # Calculate deviation from normal behavior
        # This is simplified - use Isolation Forest or similar in production
        anomaly_indicators = []
        
        if metrics.login_frequency > 20:  # Unusually high
            anomaly_indicators.append("Excessive login attempts")
        
        if metrics.working_hours_per_day > 18:
            anomaly_indicators.append("Unusual working hours")
        
        if metrics.tickets_raised > 50:
            anomaly_indicators.append("Abnormal ticket creation rate")
        
        is_anomalous = len(anomaly_indicators) > 0
        
        return {
            "is_anomalous": is_anomalous,
            "anomaly_score": min(len(anomaly_indicators) * 0.3, 1.0),
            "indicators": anomaly_indicators
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def extract_text_features(text: str):
    """
    Extract features from text for classification
    Simplified version - use proper TF-IDF or embeddings in production
    """
    # Placeholder for text vectorization
    # In production, use sklearn TfidfVectorizer or transformers
    return [len(text), text.count(' '), len(set(text.lower().split()))]

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Frontend React Components

### User Dashboard

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import KanbanBoard from './KanbanBoard';
import TimeTracker from './TimeTracker';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [tickets, setTickets] = useState([]);
  const [metrics, setMetrics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      const [tasksRes, ticketsRes] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/api/tickets`, config)
      ]);

      setTasks(tasksRes.data.data);
      setTickets(ticketsRes.data.data);
      
      // Fetch AI insights
      await fetchAIInsights(tasksRes.data.data, config);
      
      setLoading(false);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
      setLoading(false);
    }
  };

  const fetchAIInsights = async (userTasks, config) => {
    try {
      const completedTasks = userTasks.filter(t => t.status === 'done').length;
      const pendingTasks = userTasks.filter(t => t.status !== 'done').length;
      const avgTime = userTasks.reduce((sum, t) => sum + (t.timeTracked || 0), 0) / userTasks.length || 0;

      const metricsPayload = {
        user_id: localStorage.getItem('userId'),
        tasks_completed: completedTasks,
        tasks_pending: pendingTasks,
        avg_task_time: avgTime,
        tickets_raised: tickets.length,
        login_frequency: 5, // Calculate based on login history
        working_hours_per_day: 8
      };

      const burnoutRes = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/detect-burnout`,
        metricsPayload
      );

      setMetrics(burnoutRes.data);
    } catch (error) {
      console.error('Error fetching AI insights:', error);
    }
  };

  const handleTaskUpdate = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.put(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      fetchDashboardData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="user-dashboard">
      <h1>My Dashboard</h1>
      
      {metrics && metrics.burnout_risk === 'high' && (
        <div className="alert alert-warning">
          <h3>⚠️ Burnout Risk Detected</h3>
          <p>Recommendations:</p>
          <ul>
            {metrics.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      )}

      <div className="dashboard-stats">
        <div className="stat-card">
          <h3>Tasks Completed</h3>
          <p>{tasks.filter(t => t.status === 'done').length}</p>
        </div>
        <div className="stat-card">
          <h3>Pending Tasks</h3>
          <p>{tasks.filter(t => t.status !== 'done').length}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p>{tickets.filter(t => t.status === 'open').length}</p>
        </div>
      </div>

      <KanbanBoard tasks={tasks} onTaskUpdate={handleTaskUpdate} />
      
      <TimeTracker tasks={tasks.filter(t => t.status === 'in-progress')} />
    </div>
  );
};

export default UserDashboard;
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React from 'react';
import './KanbanBoard.css';

const KanbanBoard = ({ tasks, onTaskUpdate }) => {
  const columns = {
    todo: tasks.filter(t => t.status === 'todo'),
    'in-progress': tasks.filter(t => t.status === 'in-progress'),
    done: tasks.filter(t => t.status === 'done')
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = (e, status) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    onTaskUpdate(taskId, status);
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  return (
    <div className="kanban-board">
      {Object.keys(columns).map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{status.toUpperCase().replace('-', ' ')}</h3>
          <div className="task-list">
            {columns[status].map(task => (
              <div
                key={task._id}
                className="task-card"
                draggable
                onDragStart={(e) => handleDragStart(e, task._id)}
              >
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <div className="task-meta">
                  <span className={`priority priority-${task.priority}`}>
                    {task.priority}
                  </span>
                  {task.timeTracked > 0 && (
                    <span className="time-tracked">
                      {Math.floor(task.timeTracked / 60)}h {task.timeTracked % 60}m
                    </span>
                  )}
                </div>
              </div>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);

  useEffect(() => {
    fetchAdminData();
  }, []);

  const fetchAdminData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      const usersRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/admin/users`,
        config
      );
      
      setUsers(usersRes.data.data);
      
      // Analyze users for risk
      await analyzeUserRisks(usersRes.data.data, config);
    } catch (error) {
      console.error('Error fetching admin data:', error);
    }
  };

  const analyzeUserRisks = async (users, config) => {
    try {
      const riskAnalysis = await Promise.all(
        users.map(async (user) => {
          const tasksRes = await axios.get(
            `${process.env.REACT_APP_API_URL}/api/admin/users/${user._id}/tasks`,
            config
          );
          
          const tasks = tasksRes.data.data;
          const metrics = {
            user_id: user._id,
            tasks_completed: tasks.filter(t => t.status === 'done').length,
            tasks_pending: tasks.filter(t => t.status !== 'done').length,
            avg_task_time: tasks.reduce((sum, t) => sum + (t.timeTracked || 0), 0) / tasks.length || 0,
            tickets_raised: user.ticketCount || 0,
            login_frequency: user.loginFrequency || 5,
            working_hours_per_day: user.avgWorkingHours || 8
          };

          const riskRes = await axios.post(
            `${process.env.REACT_APP_ML_API_URL}/predict-risk`,
            metrics
          );

          return {
            ...user,
            riskData: riskRes.data
          };
        })
      );

      const highRiskUsers = riskAnalysis.filter(u => u.riskData.risk_level === 'high');
      setRiskUsers(highRiskUsers);
    } catch (error) {
      console.error('Error analyzing user risks:', error);
    }
  };

  const handleDeleteUser = async (userId) => {
    if (!window.confirm('Are you sure you want to delete this user?')) return;

    try {
      const token = localStorage.getItem('token');
      await axios.delete(
        `${process.env.REACT_APP_API_URL}/api/admin/users/${userId}`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      fetchAdminData();
    } catch (error) {
      console.error('Error deleting user:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>

      {riskUsers.length > 0 && (
        <div className="alert alert-danger">
          <h3>🚨 High Risk Users Detected</h3>
          <ul>
            {riskUsers.map(user => (
              <li key={user._id}>
                {user.username} - Risk Score: {(user.riskData.risk_score * 100).toFixed(1)}%
                <ul>
                  {user.riskData.factors.map((factor, idx) => (
                    <li key={idx}>{factor}</li>
                  ))}
                </ul>
              </li>
            ))}
          </ul>
        </div>
      )}

      <div className="users-table">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Username</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.username}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>
                  {user.riskData && (
                    <span className={`risk-badge risk-${user.riskData.risk_level}`}>
                      {user.riskData.risk_level} risk
                    </span>
                  )}
                </td>
                <td>
                  <button onClick={() => handleDeleteUser(user._id)}>Delete</button>
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

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const UserSchema = new mongoose.Schema({
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
    lowercase: true
  },
  password: {
    type: String,
