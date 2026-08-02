---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, task tracking, and ticket classification using React, Node.js, and FastAPI ML services
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "integrate ML service for ticket classification"
  - "build admin panel with AI insights"
  - "add burnout detection to user system"
  - "configure JWT authentication for user management"
  - "deploy user management system with AI features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines traditional user/task management with intelligent AI-powered insights. The system provides role-based access control, Kanban-style task tracking, support ticket management, and ML-driven analytics including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

**Architecture:**
- **Frontend**: React.js (port 3000)
- **Backend**: Node.js with Express (port 5000)
- **ML Service**: FastAPI with scikit-learn and River (port 8000)
- **Database**: MongoDB
- **Authentication**: JWT-based

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
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

npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file for ML service
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

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

npm start
```

## Key Components

### Authentication Flow

**Backend - User Registration (Node.js)**

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

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
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

**Frontend - Login Component (React)**

```javascript
// frontend/src/components/Auth/Login.js
import React, { useState } from 'react';
import axios from 'axios';

const Login = ({ onLogin }) => {
  const [formData, setFormData] = useState({
    email: '',
    password: ''
  });
  const [error, setError] = useState('');

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_API_URL}/api/auth/login`,
        formData
      );
      
      // Store token
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
      
      // Set axios default header
      axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
      
      onLogin(response.data.user);
    } catch (err) {
      setError(err.response?.data?.message || 'Login failed');
    }
  };

  return (
    <div className="login-container">
      <form onSubmit={handleSubmit}>
        <input
          type="email"
          placeholder="Email"
          value={formData.email}
          onChange={(e) => setFormData({ ...formData, email: e.target.value })}
          required
        />
        <input
          type="password"
          placeholder="Password"
          value={formData.password}
          onChange={(e) => setFormData({ ...formData, password: e.target.value })}
          required
        />
        {error && <div className="error">{error}</div>}
        <button type="submit">Login</button>
      </form>
    </div>
  );
};

export default Login;
```

### Task Management

**Backend - Task API (Node.js)**

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

// Get tasks for logged-in user
router.get('/my-tasks', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'name email')
      .sort({ createdAt: -1 });
    
    res.json({ success: true, tasks });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Create task (Admin only)
router.post('/', [authMiddleware, adminMiddleware], async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      assignedBy: req.user.id,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });
    
    await task.save();
    
    res.status(201).json({ success: true, task });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    // Check permission
    if (task.assignedTo.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Not authorized' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    
    res.json({ success: true, task });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

**Frontend - Kanban Board (React)**

```javascript
// frontend/src/components/Dashboard/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks/my-tasks`
      );
      
      const categorized = {
        todo: [],
        inProgress: [],
        done: []
      };
      
      response.data.tasks.forEach(task => {
        if (task.status === 'todo') categorized.todo.push(task);
        else if (task.status === 'in_progress') categorized.inProgress.push(task);
        else if (task.status === 'done') categorized.done.push(task);
      });
      
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-badge ${task.priority}`}>
        {task.priority}
      </span>
      <div className="task-actions">
        {task.status !== 'done' && (
          <button onClick={() => updateTaskStatus(task._id, 'done')}>
            Mark Done
          </button>
        )}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI/ML Integration

**ML Service - Ticket Classification (FastAPI)**

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import pickle
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import os

app = FastAPI()

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketData(BaseModel):
    title: str
    description: str
    user_id: str

class RiskAnalysisData(BaseModel):
    user_id: str
    login_frequency: int
    failed_logins: int
    unusual_hours: int
    tasks_completed: int
    tasks_delayed: int

class BurnoutData(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_completion_time: float
    overtime_hours: float
    days_worked: int

@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketData):
    """
    Classify support ticket into categories: technical, hr, access, general
    """
    try:
        # Simple rule-based classification (replace with trained model)
        text = f"{ticket.title} {ticket.description}".lower()
        
        if any(word in text for word in ['password', 'access', 'login', 'permission']):
            category = 'access'
            priority = 'high'
        elif any(word in text for word in ['bug', 'error', 'crash', 'not working']):
            category = 'technical'
            priority = 'high'
        elif any(word in text for word in ['leave', 'salary', 'hr', 'policy']):
            category = 'hr'
            priority = 'medium'
        else:
            category = 'general'
            priority = 'low'
        
        # Route to appropriate admin
        routing = {
            'access': 'admin-it',
            'technical': 'admin-tech',
            'hr': 'admin-hr',
            'general': 'admin-general'
        }
        
        return {
            'category': category,
            'priority': priority,
            'assigned_to': routing[category],
            'confidence': 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/risk-analysis")
async def analyze_risk(data: RiskAnalysisData):
    """
    Analyze user behavior for security risks
    """
    try:
        risk_score = 0
        factors = []
        
        # Failed login attempts
        if data.failed_logins > 5:
            risk_score += 30
            factors.append('Multiple failed login attempts')
        
        # Unusual hours
        if data.unusual_hours > 10:
            risk_score += 20
            factors.append('Activity during unusual hours')
        
        # Task performance
        if data.tasks_delayed > data.tasks_completed * 0.5:
            risk_score += 15
            factors.append('High task delay rate')
        
        # Login frequency
        if data.login_frequency > 50:
            risk_score += 10
            factors.append('Unusually high login frequency')
        
        risk_level = 'low'
        if risk_score > 50:
            risk_level = 'high'
        elif risk_score > 25:
            risk_level = 'medium'
        
        return {
            'user_id': data.user_id,
            'risk_score': min(risk_score, 100),
            'risk_level': risk_level,
            'factors': factors,
            'recommendation': 'Review account activity' if risk_level == 'high' else 'Monitor'
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/burnout-detection")
async def detect_burnout(data: BurnoutData):
    """
    Detect employee burnout based on workload metrics
    """
    try:
        burnout_score = 0
        indicators = []
        
        # High task assignment
        if data.tasks_assigned > 20:
            burnout_score += 25
            indicators.append('High task volume')
        
        # Low completion rate
        completion_rate = data.tasks_completed / data.tasks_assigned if data.tasks_assigned > 0 else 1
        if completion_rate < 0.7:
            burnout_score += 20
            indicators.append('Low completion rate')
        
        # Overtime hours
        if data.overtime_hours > 20:
            burnout_score += 30
            indicators.append('Excessive overtime')
        
        # Slow completion time
        if data.avg_completion_time > 5:  # days
            burnout_score += 15
            indicators.append('Slow task completion')
        
        # Continuous work without breaks
        if data.days_worked > 25:
            burnout_score += 10
            indicators.append('No breaks taken')
        
        burnout_level = 'low'
        if burnout_score > 60:
            burnout_level = 'high'
        elif burnout_score > 35:
            burnout_level = 'medium'
        
        return {
            'user_id': data.user_id,
            'burnout_score': min(burnout_score, 100),
            'burnout_level': burnout_level,
            'indicators': indicators,
            'recommendation': 'Immediate intervention needed' if burnout_level == 'high' 
                            else 'Monitor workload' if burnout_level == 'medium' 
                            else 'Continue monitoring'
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

**Backend - ML Integration (Node.js)**

```javascript
// backend/services/mlService.js
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL || 'http://localhost:8000';

class MLService {
  async classifyTicket(ticketData) {
    try {
      const response = await axios.post(
        `${ML_SERVICE_URL}/classify-ticket`,
        ticketData
      );
      return response.data;
    } catch (error) {
      console.error('ML Classification error:', error);
      return { category: 'general', priority: 'medium', assigned_to: 'admin-general' };
    }
  }

  async analyzeUserRisk(userId, userData) {
    try {
      const response = await axios.post(
        `${ML_SERVICE_URL}/risk-analysis`,
        { user_id: userId, ...userData }
      );
      return response.data;
    } catch (error) {
      console.error('Risk analysis error:', error);
      return { risk_level: 'unknown', risk_score: 0 };
    }
  }

  async detectBurnout(userId, workloadData) {
    try {
      const response = await axios.post(
        `${ML_SERVICE_URL}/burnout-detection`,
        { user_id: userId, ...workloadData }
      );
      return response.data;
    } catch (error) {
      console.error('Burnout detection error:', error);
      return { burnout_level: 'unknown', burnout_score: 0 };
    }
  }
}

module.exports = new MLService();
```

**Backend - Ticket Creation with AI (Node.js)**

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const mlService = require('../services/mlService');
const { authMiddleware } = require('../middleware/auth');

router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Get AI classification
    const classification = await mlService.classifyTicket({
      title,
      description,
      user_id: req.user.id
    });
    
    // Create ticket with AI insights
    const ticket = new Ticket({
      title,
      description,
      createdBy: req.user.id,
      category: classification.category,
      priority: classification.priority,
      assignedTo: classification.assigned_to,
      status: 'open',
      aiClassification: {
        category: classification.category,
        confidence: classification.confidence
      }
    });
    
    await ticket.save();
    
    res.status(201).json({
      success: true,
      ticket,
      aiInsights: classification
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Admin Dashboard with Analytics

**Backend - Analytics API (Node.js)**

```javascript
// backend/routes/analytics.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const Task = require('../models/Task');
const Ticket = require('../models/Ticket');
const mlService = require('../services/mlService');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

router.get('/dashboard', [authMiddleware, adminMiddleware], async (req, res) => {
  try {
    // Basic stats
    const totalUsers = await User.countDocuments({ role: 'user' });
    const totalTasks = await Task.countDocuments();
    const completedTasks = await Task.countDocuments({ status: 'done' });
    const openTickets = await Ticket.countDocuments({ status: 'open' });
    
    // User performance
    const userPerformance = await Task.aggregate([
      {
        $group: {
          _id: '$assignedTo',
          totalTasks: { $sum: 1 },
          completedTasks: {
            $sum: { $cond: [{ $eq: ['$status', 'done'] }, 1, 0] }
          },
          avgCompletionTime: { $avg: '$completionTime' }
        }
      },
      {
        $lookup: {
          from: 'users',
          localField: '_id',
          foreignField: '_id',
          as: 'user'
        }
      }
    ]);
    
    // Ticket distribution
    const ticketStats = await Ticket.aggregate([
      {
        $group: {
          _id: '$category',
          count: { $sum: 1 },
          avgResolutionTime: { $avg: '$resolutionTime' }
        }
      }
    ]);
    
    res.json({
      success: true,
      stats: {
        totalUsers,
        totalTasks,
        completedTasks,
        openTickets,
        completionRate: totalTasks > 0 ? (completedTasks / totalTasks * 100).toFixed(2) : 0
      },
      userPerformance,
      ticketStats
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

router.get('/user-risk/:userId', [authMiddleware, adminMiddleware], async (req, res) => {
  try {
    const userId = req.params.userId;
    
    // Gather user activity data
    const user = await User.findById(userId);
    const tasks = await Task.find({ assignedTo: userId });
    
    const riskData = {
      login_frequency: user.loginHistory?.length || 0,
      failed_logins: user.failedLoginAttempts || 0,
      unusual_hours: user.unusualHourLogins || 0,
      tasks_completed: tasks.filter(t => t.status === 'done').length,
      tasks_delayed: tasks.filter(t => t.dueDate < new Date() && t.status !== 'done').length
    };
    
    // Get AI risk analysis
    const riskAnalysis = await mlService.analyzeUserRisk(userId, riskData);
    
    res.json({
      success: true,
      userId,
      riskAnalysis
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

router.get('/burnout-analysis/:userId', [authMiddleware, adminMiddleware], async (req, res) => {
  try {
    const userId = req.params.userId;
    
    const tasks = await Task.find({ assignedTo: userId });
    const user = await User.findById(userId);
    
    const burnoutData = {
      tasks_assigned: tasks.length,
      tasks_completed: tasks.filter(t => t.status === 'done').length,
      avg_completion_time: tasks.reduce((sum, t) => sum + (t.completionTime || 0), 0) / tasks.length || 0,
      overtime_hours: user.overtimeHours || 0,
      days_worked: user.daysWorked || 0
    };
    
    // Get AI burnout detection
    const burnoutAnalysis = await mlService.detectBurnout(userId, burnoutData);
    
    res.json({
      success: true,
      userId,
      burnoutAnalysis
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

**Frontend - Admin Analytics Dashboard (React)**

```javascript
// frontend/src/components/Admin/Analytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { Chart as ChartJS, ArcElement, Tooltip, Legend, CategoryScale, LinearScale, BarElement } from 'chart.js';
import { Pie, Bar } from 'react-chartjs-2';

ChartJS.register(ArcElement, Tooltip, Legend, CategoryScale, LinearScale, BarElement);

const Analytics = () => {
  const [dashboardData, setDashboardData] = useState(null);
  const [selectedUser, setSelectedUser] = useState(null);
  const [riskData, setRiskData] = useState(null);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/analytics/dashboard`
      );
      setDashboardData(response.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const fetchUserRisk = async (userId) => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/analytics/user-risk/${userId}`
      );
      setRiskData(response.data.riskAnalysis);
    } catch (error) {
      console.error('Error fetching risk data:', error);
    }
  };

  if (!dashboardData) return <div>Loading...</div>;

  const ticketChartData = {
    labels: dashboardData.ticketStats.map(t => t._id),
    datasets: [{
      label: 'Tickets by Category',
      data: dashboardData.ticketStats.map(t => t.count),
      backgroundColor: ['#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0']
    }]
  };

  const performanceChartData = {
    labels: dashboardData.userPerformance.map(u => u.user[0]?.name || 'Unknown'),
    datasets: [{
      label: 'Tasks Completed',
      data: dashboardData.userPerformance.map(u => u.completedTasks),
      backgroundColor: '#36A2EB'
    }]
  };

  return (
    <div className="analytics-dashboard">
      <h1>Analytics Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-value">{dashboardData.stats.totalUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Total Tasks</h3>
          <p className="stat-value">{dashboardData.stats.totalTasks}</p>
        </div>
        <div className="stat-card">
          <h3>Completion Rate</h3>
          <p className="stat-value">{dashboardData.stats.completionRate}%</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p className="stat-value">{dashboardData.stats.openTickets}</p>
        </div>
      </div>

      <div className="charts-grid">
        <div className="chart-container">
          <h3>Ticket Distribution</h3>
          <Pie data={ticketChartData} />
        </div>
        <div className="chart-container">
          <h3>User Performance</h3>
          <Bar data={performanceChartData} />
        </div>
      </div>

      {riskData && (
        <div className="risk-analysis">
          <h3>Risk Analysis</h3>
          <div className={`risk-badge ${riskData.risk_level}`}>
            Risk Level: {riskData.risk_level} ({riskData.risk_score}/100)
          </div>
          <ul>
            {riskData.factors.map((factor, idx) => (
              <li key={idx}>{factor}</li>
            ))}
          </ul>
          <p><strong>Recommendation:</strong> {riskData.recommendation}</p>
        </div>
      )}
    </div>
  );
};

export default Analytics;
```

## Database Models

**User Model (MongoDB/Mongoose)**

```javascript
// backend/models/User.js
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
  loginHistory: [{
    timestamp: Date,
    ipAddress: String
  }],
  failedLoginAttempts: {
    type: Number,
    default: 0
  },
  unusualHourLogins: {
    type: Number,
    default: 0
  },
  overtimeHours: {
    type: Number,
    default: 0
  },
  daysWorked: {
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

**Task Model**

```javascript
// backend/models/Task.js
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
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  status
