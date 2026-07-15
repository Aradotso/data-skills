---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management capabilities
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "show me how to implement user task tracking with AI"
  - "I need to build a user management dashboard with risk detection"
  - "how do I use the AI ticket classification system"
  - "help me configure the ML service for user analytics"
  - "show me how to implement burnout detection for users"
  - "I want to add AI-powered insights to my admin dashboard"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a comprehensive full-stack application for enterprise user management that combines traditional CRUD operations with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights. The system features role-based access control, Kanban task management, support ticket handling, and real-time notifications.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Complete CRUD operations with role-based access control
- **Task Management**: Kanban board with time tracking and progress monitoring
- **Support Tickets**: AI-powered classification and routing system
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Admin Dashboard**: Comprehensive analytics and monitoring tools
- **Security**: JWT authentication with audit logs and suspicious activity detection

## Architecture

The system consists of three main components:

1. **Frontend** (React.js): User interface for both admin and regular users
2. **Backend** (Node.js): REST API handling business logic and data management
3. **ML Service** (FastAPI): AI/ML models for intelligent analytics and predictions

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB 4.4+

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
NODE_ENV=development
EOF

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
LOG_LEVEL=info
EOF

uvicorn main:app --reload --host 0.0.0.0 --port 8000
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

## Backend API Reference

### Authentication Endpoints

```javascript
// User Registration
POST /api/auth/register
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
}

// User Login
POST /api/auth/login
Content-Type: application/json

{
  "email": "john@example.com",
  "password": "securePassword123"
}

// Response includes JWT token
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "507f1f77bcf86cd799439011",
    "name": "John Doe",
    "email": "john@example.com",
    "role": "user"
  }
}
```

### User Management Endpoints

```javascript
// Get All Users (Admin only)
GET /api/users
Authorization: Bearer ${JWT_TOKEN}

// Get User by ID
GET /api/users/:id
Authorization: Bearer ${JWT_TOKEN}

// Update User
PUT /api/users/:id
Authorization: Bearer ${JWT_TOKEN}
Content-Type: application/json

{
  "name": "John Smith",
  "role": "admin",
  "status": "active"
}

// Delete User (Admin only)
DELETE /api/users/:id
Authorization: Bearer ${JWT_TOKEN}
```

### Task Management Endpoints

```javascript
// Create Task
POST /api/tasks
Authorization: Bearer ${JWT_TOKEN}
Content-Type: application/json

{
  "title": "Implement user dashboard",
  "description": "Create responsive dashboard with analytics",
  "assignedTo": "507f1f77bcf86cd799439011",
  "priority": "high",
  "dueDate": "2026-05-01",
  "estimatedHours": 16
}

// Update Task Status
PATCH /api/tasks/:id/status
Authorization: Bearer ${JWT_TOKEN}
Content-Type: application/json

{
  "status": "in_progress"
}

// Track Time
POST /api/tasks/:id/time
Authorization: Bearer ${JWT_TOKEN}
Content-Type: application/json

{
  "hours": 2.5,
  "description": "Implemented user authentication flow"
}

// Get User Tasks
GET /api/tasks/user/:userId
Authorization: Bearer ${JWT_TOKEN}
```

### Support Ticket Endpoints

```javascript
// Create Ticket
POST /api/tickets
Authorization: Bearer ${JWT_TOKEN}
Content-Type: application/json

{
  "title": "Login issue with SSO",
  "description": "Unable to authenticate using company SSO",
  "priority": "high",
  "category": "technical"
}

// Get Tickets
GET /api/tickets
Authorization: Bearer ${JWT_TOKEN}

// Update Ticket
PATCH /api/tickets/:id
Authorization: Bearer ${JWT_TOKEN}
Content-Type: application/json

{
  "status": "resolved",
  "resolution": "Updated SSO configuration"
}
```

## ML Service API Reference

### AI Analytics Endpoints

```python
# Risk Prediction
POST /api/ml/predict-risk
Content-Type: application/json

{
  "user_id": "507f1f77bcf86cd799439011",
  "features": {
    "failed_login_attempts": 3,
    "login_hours_anomaly": 2.5,
    "unusual_activity_score": 0.7,
    "task_completion_rate": 0.65,
    "average_response_time": 120
  }
}

# Response
{
  "risk_score": 0.73,
  "risk_level": "high",
  "factors": [
    "Multiple failed login attempts",
    "Unusual activity hours",
    "Low task completion rate"
  ],
  "recommendations": [
    "Enable two-factor authentication",
    "Review recent activity logs"
  ]
}
```

```python
# Burnout Detection
POST /api/ml/detect-burnout
Content-Type: application/json

{
  "user_id": "507f1f77bcf86cd799439011",
  "metrics": {
    "hours_worked_weekly": 55,
    "tasks_completed": 12,
    "tasks_overdue": 5,
    "overtime_hours": 15,
    "days_without_break": 14,
    "stress_indicators": {
      "late_responses": 8,
      "missed_deadlines": 3
    }
  }
}

# Response
{
  "burnout_score": 0.82,
  "risk_level": "critical",
  "indicators": [
    "Excessive overtime (15+ hours)",
    "No breaks for 14 days",
    "Multiple missed deadlines"
  ],
  "recommendations": [
    "Reduce workload by 30%",
    "Schedule mandatory time off",
    "Redistribute 3 tasks to other team members"
  ]
}
```

```python
# Ticket Classification
POST /api/ml/classify-ticket
Content-Type: application/json

{
  "title": "Cannot access database from application",
  "description": "Getting connection timeout errors when trying to query the user database. Error code: ETIMEDOUT",
  "metadata": {
    "user_role": "developer",
    "environment": "production"
  }
}

# Response
{
  "category": "technical",
  "priority": "high",
  "suggested_assignee": "database_team",
  "estimated_resolution_time": "2-4 hours",
  "confidence": 0.89,
  "similar_tickets": [
    "ticket_12345",
    "ticket_12389"
  ]
}
```

```python
# Anomaly Detection
POST /api/ml/detect-anomaly
Content-Type: application/json

{
  "user_id": "507f1f77bcf86cd799439011",
  "activity": {
    "timestamp": "2026-04-15T03:30:00Z",
    "action": "bulk_data_export",
    "ip_address": "192.168.1.100",
    "location": "Unknown",
    "data_volume": 50000
  }
}

# Response
{
  "is_anomaly": true,
  "anomaly_score": 0.91,
  "reasons": [
    "Unusual access time (3:30 AM)",
    "Large data export (50k records)",
    "Access from new IP address",
    "Location mismatch"
  ],
  "action_required": "immediate_review"
}
```

```python
# Project Delay Prediction
POST /api/ml/predict-delay
Content-Type: application/json

{
  "project_id": "proj_123",
  "metrics": {
    "tasks_total": 50,
    "tasks_completed": 20,
    "days_elapsed": 30,
    "days_remaining": 20,
    "team_velocity": 0.67,
    "blockers_count": 5,
    "critical_path_delay": 3
  }
}

# Response
{
  "delay_probability": 0.78,
  "estimated_delay_days": 12,
  "risk_factors": [
    "Current velocity below target (0.67 vs 1.0)",
    "5 active blockers",
    "Critical path delayed by 3 days"
  ],
  "mitigation_strategies": [
    "Add 2 more team members",
    "Resolve critical blockers first",
    "Reduce scope by 15%"
  ]
}
```

## Frontend Integration Examples

### React Component for User Dashboard

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [tickets, setTickets] = useState([]);
  const [analytics, setAnalytics] = useState(null);
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

      const [tasksRes, ticketsRes, analyticsRes] = await Promise.all([
        axios.get(`${API_URL}/api/tasks/my-tasks`, config),
        axios.get(`${API_URL}/api/tickets/my-tickets`, config),
        axios.get(`${API_URL}/api/analytics/user`, config)
      ]);

      setTasks(tasksRes.data.tasks);
      setTickets(ticketsRes.data.tickets);
      setAnalytics(analyticsRes.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchDashboardData(); // Refresh data
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p>{tasks.filter(t => t.status !== 'done').length}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p>{tickets.filter(t => t.status === 'open').length}</p>
        </div>
        <div className="stat-card">
          <h3>Completion Rate</h3>
          <p>{analytics?.completionRate || 0}%</p>
        </div>
      </div>

      <div className="kanban-board">
        <div className="column">
          <h3>To Do</h3>
          {tasks.filter(t => t.status === 'todo').map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => updateTaskStatus(task._id, 'in_progress')}>
                Start
              </button>
            </div>
          ))}
        </div>
        
        <div className="column">
          <h3>In Progress</h3>
          {tasks.filter(t => t.status === 'in_progress').map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => updateTaskStatus(task._id, 'done')}>
                Complete
              </button>
            </div>
          ))}
        </div>
        
        <div className="column">
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
  );
};

export default UserDashboard;
```

### AI Analytics Integration Component

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const AIAnalytics = ({ userId }) => {
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAIInsights();
  }, [userId]);

  const fetchAIInsights = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      // Fetch user metrics first
      const metricsRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/analytics/user/${userId}/metrics`,
        config
      );

      const metrics = metricsRes.data;

      // Risk prediction
      const riskRes = await axios.post(
        `${ML_API_URL}/api/ml/predict-risk`,
        {
          user_id: userId,
          features: {
            failed_login_attempts: metrics.failedLogins || 0,
            login_hours_anomaly: metrics.loginAnomalyScore || 0,
            unusual_activity_score: metrics.unusualActivityScore || 0,
            task_completion_rate: metrics.taskCompletionRate || 1.0,
            average_response_time: metrics.avgResponseTime || 60
          }
        }
      );

      // Burnout detection
      const burnoutRes = await axios.post(
        `${ML_API_URL}/api/ml/detect-burnout`,
        {
          user_id: userId,
          metrics: {
            hours_worked_weekly: metrics.weeklyHours || 40,
            tasks_completed: metrics.tasksCompleted || 0,
            tasks_overdue: metrics.tasksOverdue || 0,
            overtime_hours: metrics.overtimeHours || 0,
            days_without_break: metrics.daysWithoutBreak || 0,
            stress_indicators: {
              late_responses: metrics.lateResponses || 0,
              missed_deadlines: metrics.missedDeadlines || 0
            }
          }
        }
      );

      setRiskData(riskRes.data);
      setBurnoutData(burnoutRes.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching AI insights:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Analyzing data...</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>

      {/* Risk Analysis */}
      <div className={`insight-card risk-${riskData.risk_level}`}>
        <h3>Security Risk Analysis</h3>
        <div className="risk-score">
          Risk Score: {(riskData.risk_score * 100).toFixed(0)}%
        </div>
        <div className="risk-level">
          Level: <span>{riskData.risk_level.toUpperCase()}</span>
        </div>
        <div className="factors">
          <h4>Risk Factors:</h4>
          <ul>
            {riskData.factors.map((factor, idx) => (
              <li key={idx}>{factor}</li>
            ))}
          </ul>
        </div>
        <div className="recommendations">
          <h4>Recommendations:</h4>
          <ul>
            {riskData.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      </div>

      {/* Burnout Analysis */}
      <div className={`insight-card burnout-${burnoutData.risk_level}`}>
        <h3>Burnout Risk Analysis</h3>
        <div className="burnout-score">
          Burnout Score: {(burnoutData.burnout_score * 100).toFixed(0)}%
        </div>
        <div className="risk-level">
          Level: <span>{burnoutData.risk_level.toUpperCase()}</span>
        </div>
        <div className="indicators">
          <h4>Warning Indicators:</h4>
          <ul>
            {burnoutData.indicators.map((indicator, idx) => (
              <li key={idx}>{indicator}</li>
            ))}
          </ul>
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
  );
};

export default AIAnalytics;
```

### Ticket Creation with AI Classification

```javascript
import React, { useState } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const TicketForm = ({ onTicketCreated }) => {
  const [formData, setFormData] = useState({
    title: '',
    description: '',
    priority: 'medium'
  });
  const [aiSuggestions, setAiSuggestions] = useState(null);
  const [loading, setLoading] = useState(false);

  const handleChange = (e) => {
    setFormData({
      ...formData,
      [e.target.name]: e.target.value
    });
  };

  const getAIClassification = async () => {
    if (!formData.title || !formData.description) return;

    try {
      setLoading(true);
      const response = await axios.post(
        `${ML_API_URL}/api/ml/classify-ticket`,
        {
          title: formData.title,
          description: formData.description,
          metadata: {
            user_role: localStorage.getItem('userRole') || 'user',
            environment: 'production'
          }
        }
      );

      setAiSuggestions(response.data);
      setLoading(false);
    } catch (error) {
      console.error('Error getting AI classification:', error);
      setLoading(false);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();

    try {
      const token = localStorage.getItem('token');
      const ticketData = {
        ...formData,
        category: aiSuggestions?.category || 'general',
        priority: aiSuggestions?.priority || formData.priority,
        suggestedAssignee: aiSuggestions?.suggested_assignee
      };

      const response = await axios.post(
        `${API_URL}/api/tickets`,
        ticketData,
        { headers: { Authorization: `Bearer ${token}` } }
      );

      onTicketCreated(response.data.ticket);
      setFormData({ title: '', description: '', priority: 'medium' });
      setAiSuggestions(null);
    } catch (error) {
      console.error('Error creating ticket:', error);
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
          onClick={getAIClassification}
          disabled={loading || !formData.title || !formData.description}
        >
          {loading ? 'Analyzing...' : 'Get AI Suggestions'}
        </button>

        {aiSuggestions && (
          <div className="ai-suggestions">
            <h3>AI Classification Results</h3>
            <p><strong>Category:</strong> {aiSuggestions.category}</p>
            <p><strong>Suggested Priority:</strong> {aiSuggestions.priority}</p>
            <p><strong>Estimated Resolution:</strong> {aiSuggestions.estimated_resolution_time}</p>
            <p><strong>Confidence:</strong> {(aiSuggestions.confidence * 100).toFixed(0)}%</p>
            {aiSuggestions.similar_tickets?.length > 0 && (
              <div>
                <p><strong>Similar Tickets:</strong></p>
                <ul>
                  {aiSuggestions.similar_tickets.map((ticket, idx) => (
                    <li key={idx}>{ticket}</li>
                  ))}
                </ul>
              </div>
            )}
          </div>
        )}

        <button type="submit" className="submit-btn">
          Create Ticket
        </button>
      </form>
    </div>
  );
};

export default TicketForm;
```

## Backend Implementation Patterns

### Middleware for Authentication

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (
    req.headers.authorization &&
    req.headers.authorization.startsWith('Bearer')
  ) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({
      success: false,
      message: 'Not authorized to access this route'
    });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id);
    next();
  } catch (error) {
    return res.status(401).json({
      success: false,
      message: 'Not authorized to access this route'
    });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        message: `User role ${req.user.role} is not authorized to access this route`
      });
    }
    next();
  };
};
```

### User Model with Schema

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please add a name'],
    trim: true
  },
  email: {
    type: String,
    required: [true, 'Please add an email'],
    unique: true,
    match: [
      /^\w+([\.-]?\w+)*@\w+([\.-]?\w+)*(\.\w{2,3})+$/,
      'Please add a valid email'
    ]
  },
  password: {
    type: String,
    required: [true, 'Please add a password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  department: String,
  position: String,
  profileImage: String,
  lastLogin: Date,
  loginAttempts: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Encrypt password using bcrypt
UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) {
    next();
  }
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

// Sign JWT and return
UserSchema.methods.getSignedJwtToken = function() {
  return jwt.sign({ id: this._id }, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_EXPIRE
  });
};

// Match user entered password to hashed password in database
UserSchema.methods.matchPassword = async function(enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

### Task Controller

```javascript
// controllers/taskController.js
const Task = require('../models/Task');
const User = require('../models/User');
const axios = require('axios');

// @desc    Create new task
// @route   POST /api/tasks
// @access  Private
exports.createTask = async (req, res) => {
  try {
    req.body.createdBy = req.user.id;

    const task = await Task.create(req.body);

    // Send notification to assigned user
    if (req.body.assignedTo) {
      await sendNotification(req.body.assignedTo, {
        type: 'task_assigned',
        message: `New task assigned: ${task.title}`,
        taskId: task._id
      });
    }

    res.status(201).json({
      success: true,
      task
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      error: error.message
    });
  }
};

// @desc    Get user's tasks
// @route   GET /api/tasks/my-tasks
// @access  Private
exports.getMyTasks = async (req, res) => {
  try {
    const tasks = await Task.find({
      $or: [
        { assignedTo: req.user.id },
        { createdBy: req.user.id }
      ]
    })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort('-createdAt');

    res.status(200).json({
      success: true,
      count: tasks.length,
      tasks
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      error: error.message
    });
  }
};

// @desc    Update task status
// @route   PATCH /api/tasks/:id/status
// @access  Private
exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({
        success: false,
        error: 'Task not found'
      });
    }

    // Check if user is authorized
    if (
      task.assignedTo.toString() !== req.user.id &&
      task.createdBy.toString() !== req.user.id &&
      req.user.role !== 'admin'
    ) {
      return res.status(403).json({
        success: false,
        error: 'Not authorized to update this task'
      });
    }

    task.status = req.body.status;
    task.statusHistory.push({
      status: req.body.status,
      changedBy: req.user.id,
      changedAt: Date.now()
    });

    if (req.body.status === 'done') {
      task.completedAt = Date.now();
    }

    await task.save();

    res.status(200).json({
      success: true,
      task
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      error: error.message
    });
  }
};

// @desc    Track time on task
// @route   POST /api/tasks/:id/time
// @access  Private
exports.trackTime = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({
        success: false,
        error: 
