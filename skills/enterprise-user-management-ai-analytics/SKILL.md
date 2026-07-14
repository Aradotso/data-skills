---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and intelligent ticket routing
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "show me how to implement ticket classification with AI"
  - "configure risk detection and anomaly detection features"
  - "build a user dashboard with task tracking and kanban board"
  - "implement JWT authentication for enterprise system"
  - "set up AI-powered burnout detection"
  - "create admin dashboard with user analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript/Python application that provides centralized user management, task tracking, support ticket management, and AI-powered insights. It combines React frontend, Node.js backend, MongoDB database, and FastAPI ML service to deliver risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB running locally or remote connection
- Git

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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend server
npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

ML service runs at `http://localhost:8000`

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

Frontend runs at `http://localhost:3000`

## Architecture

### Component Structure
- **Frontend (React)**: User interface with admin and user dashboards
- **Backend (Node.js/Express)**: REST API for user, task, and ticket management
- **ML Service (FastAPI)**: AI analytics endpoints for predictions and insights
- **Database (MongoDB)**: Centralized data storage

## Key Features and API Endpoints

### Authentication

#### Register User
```javascript
// Backend: routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
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
      name,
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
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});
```

#### Login User
```javascript
// Frontend: src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const login = async (email, password) => {
  try {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    
    return response.data;
  } catch (error) {
    throw error.response.data;
  }
};

export const logout = () => {
  localStorage.removeItem('token');
  localStorage.removeItem('user');
};

export const getCurrentUser = () => {
  return JSON.parse(localStorage.getItem('user'));
};
```

### User Management

#### Get All Users (Admin)
```javascript
// Backend: routes/users.js
const { protect, authorize } = require('../middleware/auth');

router.get('/', protect, authorize('admin'), async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({
      success: true,
      count: users.length,
      data: users
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});
```

#### Update User
```javascript
// Frontend: src/components/Admin/UserManagement.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserManagement = () => {
  const [users, setUsers] = useState([]);
  const API_URL = process.env.REACT_APP_API_URL;
  
  const updateUser = async (userId, updates) => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.put(
        `${API_URL}/api/users/${userId}`,
        updates,
        {
          headers: {
            Authorization: `Bearer ${token}`
          }
        }
      );
      
      setUsers(users.map(u => u._id === userId ? response.data.data : u));
      alert('User updated successfully');
    } catch (error) {
      console.error('Update error:', error);
      alert('Failed to update user');
    }
  };
  
  return (
    <div className="user-management">
      <h2>User Management</h2>
      {/* User table and edit forms */}
    </div>
  );
};
```

### Task Management

#### Create Task
```javascript
// Backend: models/Task.js
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
    default: 0 // in minutes
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  }
}, { timestamps: true });

module.exports = mongoose.model('Task', TaskSchema);
```

#### Kanban Board Component
```javascript
// Frontend: src/components/User/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });
  const API_URL = process.env.REACT_APP_API_URL;
  
  useEffect(() => {
    fetchTasks();
  }, []);
  
  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(`${API_URL}/api/tasks/my-tasks`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      
      const grouped = {
        todo: response.data.data.filter(t => t.status === 'todo'),
        'in-progress': response.data.data.filter(t => t.status === 'in-progress'),
        done: response.data.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (error) {
      console.error('Fetch tasks error:', error);
    }
  };
  
  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.put(
        `${API_URL}/api/tasks/${taskId}`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      fetchTasks(); // Refresh board
    } catch (error) {
      console.error('Update status error:', error);
    }
  };
  
  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
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
        {tasks['in-progress'].map(task => (
          <div key={task._id} className="task-card in-progress">
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
        {tasks.done.map(task => (
          <div key={task._id} className="task-card done">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### Support Ticket System

#### Create Ticket
```javascript
// Backend: models/Ticket.js
const TicketSchema = new mongoose.Schema({
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
    predictedCategory: String,
    confidence: Number,
    suggestedPriority: String
  }
}, { timestamps: true });
```

#### Submit Ticket with AI Classification
```javascript
// Frontend: src/components/User/TicketForm.jsx
import React, { useState } from 'react';
import axios from 'axios';

const TicketForm = () => {
  const [formData, setFormData] = useState({
    title: '',
    description: '',
    category: 'general'
  });
  
  const API_URL = process.env.REACT_APP_API_URL;
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;
  
  const submitTicket = async (e) => {
    e.preventDefault();
    
    try {
      const token = localStorage.getItem('token');
      
      // Get AI classification first
      const aiResponse = await axios.post(
        `${ML_API_URL}/api/ml/classify-ticket`,
        {
          title: formData.title,
          description: formData.description
        }
      );
      
      // Create ticket with AI insights
      const ticketResponse = await axios.post(
        `${API_URL}/api/tickets`,
        {
          ...formData,
          aiClassification: aiResponse.data
        },
        {
          headers: { Authorization: `Bearer ${token}` }
        }
      );
      
      alert('Ticket submitted successfully!');
      setFormData({ title: '', description: '', category: 'general' });
    } catch (error) {
      console.error('Submit ticket error:', error);
      alert('Failed to submit ticket');
    }
  };
  
  return (
    <form onSubmit={submitTicket}>
      <input
        type="text"
        placeholder="Title"
        value={formData.title}
        onChange={(e) => setFormData({...formData, title: e.target.value})}
        required
      />
      <textarea
        placeholder="Description"
        value={formData.description}
        onChange={(e) => setFormData({...formData, description: e.target.value})}
        required
      />
      <select
        value={formData.category}
        onChange={(e) => setFormData({...formData, category: e.target.value})}
      >
        <option value="general">General</option>
        <option value="technical">Technical</option>
        <option value="billing">Billing</option>
        <option value="urgent">Urgent</option>
      </select>
      <button type="submit">Submit Ticket</button>
    </form>
  );
};
```

## AI/ML Service Implementation

### Ticket Classification

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import numpy as np

app = FastAPI()

class TicketInput(BaseModel):
    title: str
    description: str

class TicketClassification(BaseModel):
    predictedCategory: str
    confidence: float
    suggestedPriority: str

# Load or train model (simplified)
vectorizer = TfidfVectorizer(max_features=1000)
classifier = MultinomialNB()

@app.post("/api/ml/classify-ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketInput):
    try:
        # Combine title and description
        text = f"{ticket.title} {ticket.description}"
        
        # Transform text (assume model is trained)
        features = vectorizer.transform([text])
        
        # Predict category
        category_pred = classifier.predict(features)[0]
        confidence = max(classifier.predict_proba(features)[0])
        
        # Determine priority based on keywords
        urgent_keywords = ['urgent', 'critical', 'down', 'error', 'broken']
        priority = 'high' if any(kw in text.lower() for kw in urgent_keywords) else 'medium'
        
        return TicketClassification(
            predictedCategory=category_pred,
            confidence=float(confidence),
            suggestedPriority=priority
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Risk Detection

```python
# ml-service/risk_detection.py
from datetime import datetime, timedelta
from pydantic import BaseModel
from typing import List

class UserActivity(BaseModel):
    userId: str
    loginTimes: List[datetime]
    tasksCompleted: int
    failedLogins: int
    dataAccessed: int

class RiskAssessment(BaseModel):
    userId: str
    riskScore: float
    riskLevel: str
    factors: List[str]
    recommendations: List[str]

@app.post("/api/ml/detect-risk", response_model=RiskAssessment)
async def detect_risk(activity: UserActivity):
    risk_score = 0.0
    factors = []
    recommendations = []
    
    # Check unusual login times
    night_logins = sum(1 for t in activity.loginTimes if t.hour < 6 or t.hour > 22)
    if night_logins > 3:
        risk_score += 20
        factors.append("Unusual login times detected")
        recommendations.append("Review login activity")
    
    # Check failed login attempts
    if activity.failedLogins > 5:
        risk_score += 30
        factors.append("Multiple failed login attempts")
        recommendations.append("Verify account security")
    
    # Check excessive data access
    if activity.dataAccessed > 1000:
        risk_score += 25
        factors.append("High data access volume")
        recommendations.append("Monitor data downloads")
    
    # Check low productivity
    if activity.tasksCompleted < 2:
        risk_score += 15
        factors.append("Low task completion rate")
    
    # Determine risk level
    if risk_score >= 60:
        risk_level = "high"
    elif risk_score >= 30:
        risk_level = "medium"
    else:
        risk_level = "low"
    
    return RiskAssessment(
        userId=activity.userId,
        riskScore=risk_score,
        riskLevel=risk_level,
        factors=factors,
        recommendations=recommendations
    )
```

### Burnout Detection

```python
# ml-service/burnout_detection.py
class WorkloadData(BaseModel):
    userId: str
    hoursWorked: float
    tasksAssigned: int
    tasksCompleted: int
    overtimeHours: float
    daysWithoutBreak: int
    averageTaskDuration: float  # in hours

class BurnoutPrediction(BaseModel):
    userId: str
    burnoutRisk: str  # low, medium, high
    burnoutScore: float
    factors: List[str]
    recommendations: List[str]

@app.post("/api/ml/detect-burnout", response_model=BurnoutPrediction)
async def detect_burnout(workload: WorkloadData):
    score = 0.0
    factors = []
    recommendations = []
    
    # Excessive hours
    if workload.hoursWorked > 50:
        score += 30
        factors.append("Working excessive hours")
        recommendations.append("Reduce workload")
    
    # Overtime
    if workload.overtimeHours > 10:
        score += 20
        factors.append("Significant overtime")
        recommendations.append("Limit overtime hours")
    
    # No breaks
    if workload.daysWithoutBreak > 10:
        score += 25
        factors.append("No recent breaks")
        recommendations.append("Schedule time off")
    
    # Task completion rate
    completion_rate = workload.tasksCompleted / max(workload.tasksAssigned, 1)
    if completion_rate < 0.5:
        score += 15
        factors.append("Low task completion rate")
        recommendations.append("Reassess task priorities")
    
    # Long task durations
    if workload.averageTaskDuration > 8:
        score += 10
        factors.append("Tasks taking longer than expected")
        recommendations.append("Provide additional support")
    
    # Determine risk level
    if score >= 60:
        risk = "high"
    elif score >= 30:
        risk = "medium"
    else:
        risk = "low"
    
    return BurnoutPrediction(
        userId=workload.userId,
        burnoutRisk=risk,
        burnoutScore=score,
        factors=factors,
        recommendations=recommendations
    )
```

### Anomaly Detection

```python
# ml-service/anomaly_detection.py
from river import anomaly
from river import compose
from river import preprocessing

# Online learning model for anomaly detection
model = compose.Pipeline(
    preprocessing.StandardScaler(),
    anomaly.HalfSpaceTrees()
)

class UserBehavior(BaseModel):
    userId: str
    loginCount: int
    dataDownloads: int
    permissionChanges: int
    ipAddresses: int
    deviceChanges: int

class AnomalyResult(BaseModel):
    userId: str
    isAnomaly: bool
    anomalyScore: float
    timestamp: datetime

@app.post("/api/ml/detect-anomaly", response_model=AnomalyResult)
async def detect_anomaly(behavior: UserBehavior):
    # Prepare features
    features = {
        'login_count': behavior.loginCount,
        'data_downloads': behavior.dataDownloads,
        'permission_changes': behavior.permissionChanges,
        'ip_addresses': behavior.ipAddresses,
        'device_changes': behavior.deviceChanges
    }
    
    # Get anomaly score
    score = model.score_one(features)
    
    # Update model with new data
    model.learn_one(features)
    
    # Threshold for anomaly
    is_anomaly = score > 0.7
    
    return AnomalyResult(
        userId=behavior.userId,
        isAnomaly=is_anomaly,
        anomalyScore=score,
        timestamp=datetime.now()
    )
```

## Configuration

### Environment Variables

#### Backend (.env)
```bash
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt

# JWT
JWT_SECRET=your_secure_jwt_secret_key
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
```

#### ML Service (.env)
```bash
# Database
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt

# Models
MODEL_PATH=./models
TRAIN_ON_STARTUP=false

# Logging
LOG_LEVEL=INFO

# API Keys (if using external services)
# OPENAI_API_KEY=your_openai_key
```

#### Frontend (.env)
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENV=development
```

## Common Patterns

### Protected Route Middleware

```javascript
// Backend: middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ message: 'Not authorized' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Invalid token' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: 'Access forbidden' });
    }
    next();
  };
};
```

### Time Tracking Component

```javascript
// Frontend: src/components/User/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId, onTimeUpdate }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  
  useEffect(() => {
    let interval = null;
    
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(seconds => seconds + 1);
      }, 1000);
    } else if (!isRunning && seconds !== 0) {
      clearInterval(interval);
    }
    
    return () => clearInterval(interval);
  }, [isRunning, seconds]);
  
  const handleStart = () => {
    setIsRunning(true);
  };
  
  const handleStop = async () => {
    setIsRunning(false);
    const minutes = Math.floor(seconds / 60);
    await onTimeUpdate(taskId, minutes);
  };
  
  const handleReset = () => {
    setSeconds(0);
    setIsRunning(false);
  };
  
  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };
  
  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <div className="controls">
        {!isRunning ? (
          <button onClick={handleStart}>Start</button>
        ) : (
          <button onClick={handleStop}>Stop</button>
        )}
        <button onClick={handleReset}>Reset</button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

### Admin Analytics Dashboard

```javascript
// Frontend: src/components/Admin/Analytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { Chart as ChartJS, ArcElement, Tooltip, Legend, CategoryScale, LinearScale, BarElement } from 'chart.js';
import { Pie, Bar } from 'react-chartjs-2';

ChartJS.register(ArcElement, Tooltip, Legend, CategoryScale, LinearScale, BarElement);

const Analytics = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeUsers: 0,
    totalTasks: 0,
    completedTasks: 0,
    openTickets: 0,
    highRiskUsers: 0
  });
  
  const API_URL = process.env.REACT_APP_API_URL;
  
  useEffect(() => {
    fetchAnalytics();
  }, []);
  
  const fetchAnalytics = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(`${API_URL}/api/admin/analytics`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      setAnalytics(response.data.data);
    } catch (error) {
      console.error('Fetch analytics error:', error);
    }
  };
  
  const taskStatusData = {
    labels: ['Completed', 'In Progress', 'To Do'],
    datasets: [{
      data: [analytics.completedTasks, analytics.inProgressTasks, analytics.todoTasks],
      backgroundColor: ['#4CAF50', '#FFC107', '#2196F3']
    }]
  };
  
  return (
    <div className="analytics-dashboard">
      <h2>Organization Analytics</h2>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-value">{analytics.totalUsers}</p>
        </div>
        
        <div className="stat-card">
          <h3>Active Users</h3>
          <p className="stat-value">{analytics.activeUsers}</p>
        </div>
        
        <div className="stat-card">
          <h3>Tasks Completed</h3>
          <p className="stat-value">{analytics.completedTasks}</p>
        </div>
        
        <div className="stat-card alert">
          <h3>High Risk Users</h3>
          <p className="stat-value">{analytics.highRiskUsers}</p>
        </div>
      </div>
      
      <div className="charts">
        <div className="chart-container">
          <h3>Task Distribution</h3>
          <Pie data={taskStatusData} />
        </div>
      </div>
    </div>
  );
};

export default Analytics;
```

## Troubleshooting

### Common Issues

#### MongoDB Connection Error
```
Error: MongooseServerSelectionError: connect ECONNREFUSED
```

**Solution:**
- Ensure MongoDB is running: `sudo systemctl start mongod` (Linux) or start MongoDB service (Windows)
- Verify connection string in `.env` file
- Check if port 27017 is available

#### JWT Token Invalid
```
Error: JsonWebTokenError: invalid token
```

**Solution:**
```javascript
// Clear localStorage and re-login
localStorage.removeItem('token');
localStorage.removeItem('user');

// Ensure JWT_SECRET is set in backend .env
// Backend should return proper token format
```

#### CORS Issues
```
Access to XMLHttpRequest blocked by CORS policy
```

**Solution:**
```javascript
// Backend: app.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

#### ML Service Not Responding
```
Error: connect ECONNREFUSED 127.0.0.1:8000
```

**Solution:**
- Check if FastAPI is running: `uvicorn main:app --reload`
- Verify ML_SERVICE_URL in backend .env
- Check Python dependencies: `pip install -r requirements.txt`

#### Model Not Found Error
```
FileNotFoundError: [Errno 2] No such file or directory: './models/classifier.pkl'
```

**Solution:**
```python
# Train and save model first
import
