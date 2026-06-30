---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket routing, and risk detection
triggers:
  - how do I use the enterprise user management system
  - set up user management with AI analytics
  - implement task tracking with AI insights
  - configure AI-powered ticket classification
  - build user dashboard with analytics
  - integrate ML service for risk detection
  - create admin dashboard with user management
  - use the AI assistant for enterprise management
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build and customize an enterprise-grade user management system with integrated AI analytics. The system provides user/admin dashboards, task tracking with Kanban boards, support ticket management, and ML-powered features including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## What This Project Does

The Enterprise User Management System is a full-stack application that combines:
- **User Management**: JWT authentication, role-based access control, user CRUD operations
- **Task Management**: Kanban board (To Do/In Progress/Done), time tracking, assignment workflows
- **Support System**: Ticket creation, AI-based classification and routing
- **AI Analytics**: Risk detection, anomaly detection, burnout prediction, project delay forecasting
- **Dashboards**: Admin analytics dashboard, user performance insights, audit logs

**Tech Stack**: React.js frontend, Node.js/Express backend, MongoDB database, FastAPI ML service with scikit-learn

## Installation

### Prerequisites
```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.4
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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
# Runs on http://localhost:5000
```

### ML Service Setup
```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

uvicorn main:app --reload
# Runs on http://localhost:8000
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
# Runs on http://localhost:3000
```

## Key API Endpoints

### Authentication APIs
```javascript
// Backend: routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register new user
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
    
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });
    
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

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email }).select('+password');
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const isMatch = await user.matchPassword(password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });
    
    // Update last login
    user.lastLogin = Date.now();
    await user.save();
    
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

### Task Management APIs
```javascript
// Backend: routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { protect, authorize } = require('../middleware/auth');

// Get all tasks (with filters)
router.get('/', protect, async (req, res) => {
  try {
    const { status, assignedTo, priority } = req.query;
    let query = {};
    
    // Users see only their tasks, admins see all
    if (req.user.role !== 'admin') {
      query.assignedTo = req.user.id;
    } else if (assignedTo) {
      query.assignedTo = assignedTo;
    }
    
    if (status) query.status = status;
    if (priority) query.priority = priority;
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort('-createdAt');
    
    res.json({ success: true, count: tasks.length, data: tasks });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task (admin only)
router.post('/', protect, authorize('admin'), async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status, timeSpent } = req.body;
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    // Check authorization
    if (req.user.role !== 'admin' && task.assignedTo.toString() !== req.user.id) {
      return res.status(403).json({ message: 'Not authorized' });
    }
    
    task.status = status;
    if (timeSpent) task.timeSpent = (task.timeSpent || 0) + timeSpent;
    if (status === 'done') task.completedAt = Date.now();
    
    await task.save();
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Support Ticket APIs
```javascript
// Backend: routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const { protect } = require('../middleware/auth');
const axios = require('axios');

// Create ticket with AI classification
router.post('/', protect, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for classification
    let category = 'general';
    let aiPriority = priority;
    
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { title, description }
      );
      category = mlResponse.data.category;
      aiPriority = mlResponse.data.priority || priority;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }
    
    const ticket = await Ticket.create({
      title,
      description,
      priority: aiPriority,
      category,
      createdBy: req.user.id,
      status: 'open'
    });
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', protect, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .sort('-createdAt');
    
    res.json({ success: true, count: tickets.length, data: tickets });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### Ticket Classification
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import joblib
import os

app = FastAPI()

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

class TicketRequest(BaseModel):
    title: str
    description: str

class TicketResponse(BaseModel):
    category: str
    priority: str
    confidence: float

# Initialize classifier
try:
    vectorizer = joblib.load(f'{MODEL_PATH}/ticket_vectorizer.pkl')
    classifier = joblib.load(f'{MODEL_PATH}/ticket_classifier.pkl')
except FileNotFoundError:
    # Initialize with defaults
    vectorizer = TfidfVectorizer(max_features=1000)
    classifier = MultinomialNB()

@app.post("/classify-ticket", response_model=TicketResponse)
async def classify_ticket(ticket: TicketRequest):
    try:
        # Combine title and description
        text = f"{ticket.title} {ticket.description}"
        
        # Transform text
        features = vectorizer.transform([text])
        
        # Predict category
        category_pred = classifier.predict(features)[0]
        confidence = max(classifier.predict_proba(features)[0])
        
        # Determine priority based on keywords
        priority = determine_priority(text, confidence)
        
        return TicketResponse(
            category=category_pred,
            priority=priority,
            confidence=float(confidence)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def determine_priority(text: str, confidence: float) -> str:
    """Determine ticket priority based on keywords and confidence"""
    text_lower = text.lower()
    
    urgent_keywords = ['urgent', 'critical', 'down', 'broken', 'error', 'crash']
    high_keywords = ['important', 'issue', 'problem', 'bug']
    
    if any(keyword in text_lower for keyword in urgent_keywords):
        return 'urgent'
    elif any(keyword in text_lower for keyword in high_keywords):
        return 'high'
    elif confidence > 0.8:
        return 'medium'
    else:
        return 'low'
```

### Risk Detection
```python
# ml-service/risk_detection.py
from fastapi import APIRouter
from pydantic import BaseModel
from typing import List, Dict
import numpy as np
from datetime import datetime, timedelta

router = APIRouter()

class UserActivity(BaseModel):
    user_id: str
    login_times: List[str]
    tasks_completed: int
    tasks_overdue: int
    avg_task_time: float
    failed_logins: int

class RiskScore(BaseModel):
    user_id: str
    risk_level: str
    score: float
    factors: Dict[str, float]
    recommendations: List[str]

@router.post("/detect-risk", response_model=RiskScore)
async def detect_risk(activity: UserActivity):
    """Detect user risk based on behavioral patterns"""
    
    factors = {}
    recommendations = []
    
    # Factor 1: Failed login attempts
    if activity.failed_logins > 5:
        factors['failed_logins'] = 0.3
        recommendations.append("Multiple failed login attempts detected")
    elif activity.failed_logins > 2:
        factors['failed_logins'] = 0.1
    else:
        factors['failed_logins'] = 0.0
    
    # Factor 2: Overdue tasks
    overdue_ratio = activity.tasks_overdue / max(activity.tasks_completed, 1)
    if overdue_ratio > 0.5:
        factors['overdue_tasks'] = 0.25
        recommendations.append("High number of overdue tasks")
    elif overdue_ratio > 0.3:
        factors['overdue_tasks'] = 0.15
    else:
        factors['overdue_tasks'] = 0.0
    
    # Factor 3: Unusual login times
    unusual_login_score = detect_unusual_logins(activity.login_times)
    factors['unusual_logins'] = unusual_login_score
    if unusual_login_score > 0.1:
        recommendations.append("Unusual login times detected")
    
    # Factor 4: Low productivity
    if activity.avg_task_time > 8 * 3600:  # More than 8 hours per task
        factors['low_productivity'] = 0.15
        recommendations.append("Tasks taking longer than usual")
    else:
        factors['low_productivity'] = 0.0
    
    # Calculate total risk score
    total_score = sum(factors.values())
    
    # Determine risk level
    if total_score > 0.6:
        risk_level = 'high'
    elif total_score > 0.3:
        risk_level = 'medium'
    else:
        risk_level = 'low'
    
    return RiskScore(
        user_id=activity.user_id,
        risk_level=risk_level,
        score=total_score,
        factors=factors,
        recommendations=recommendations if recommendations else ["No issues detected"]
    )

def detect_unusual_logins(login_times: List[str]) -> float:
    """Detect unusual login patterns"""
    if not login_times:
        return 0.0
    
    # Parse login times
    hours = []
    for time_str in login_times:
        try:
            dt = datetime.fromisoformat(time_str)
            hours.append(dt.hour)
        except ValueError:
            continue
    
    if not hours:
        return 0.0
    
    # Check for logins during unusual hours (10 PM - 6 AM)
    unusual_hours = sum(1 for h in hours if h >= 22 or h <= 6)
    unusual_ratio = unusual_hours / len(hours)
    
    return min(unusual_ratio * 0.2, 0.2)
```

### Burnout Detection
```python
# ml-service/burnout_detection.py
from fastapi import APIRouter
from pydantic import BaseModel
from typing import List
import numpy as np

router = APIRouter()

class WorkloadData(BaseModel):
    user_id: str
    tasks_per_day: List[int]
    hours_per_day: List[float]
    missed_deadlines: int
    total_tasks: int

class BurnoutAnalysis(BaseModel):
    user_id: str
    burnout_risk: str
    risk_score: float
    indicators: List[str]
    recommendations: List[str]

@router.post("/detect-burnout", response_model=BurnoutAnalysis)
async def detect_burnout(workload: WorkloadData):
    """Analyze workload patterns to detect burnout risk"""
    
    indicators = []
    recommendations = []
    risk_factors = []
    
    # Calculate metrics
    avg_tasks = np.mean(workload.tasks_per_day) if workload.tasks_per_day else 0
    avg_hours = np.mean(workload.hours_per_day) if workload.hours_per_day else 0
    task_variance = np.std(workload.tasks_per_day) if len(workload.tasks_per_day) > 1 else 0
    
    # Factor 1: Excessive hours
    if avg_hours > 10:
        risk_factors.append(0.35)
        indicators.append("Working excessive hours consistently")
        recommendations.append("Reduce daily working hours to 8-9 hours")
    elif avg_hours > 9:
        risk_factors.append(0.2)
        indicators.append("Working above normal hours")
    
    # Factor 2: High task load
    if avg_tasks > 15:
        risk_factors.append(0.25)
        indicators.append("High daily task load")
        recommendations.append("Redistribute tasks to other team members")
    elif avg_tasks > 10:
        risk_factors.append(0.15)
    
    # Factor 3: Missed deadlines
    missed_ratio = workload.missed_deadlines / max(workload.total_tasks, 1)
    if missed_ratio > 0.3:
        risk_factors.append(0.25)
        indicators.append("Frequently missing deadlines")
        recommendations.append("Review task prioritization and deadlines")
    elif missed_ratio > 0.15:
        risk_factors.append(0.15)
    
    # Factor 4: Inconsistent workload
    if task_variance > 5:
        risk_factors.append(0.15)
        indicators.append("Highly variable workload")
        recommendations.append("Balance task distribution throughout the week")
    
    # Calculate risk score
    risk_score = sum(risk_factors)
    
    # Determine risk level
    if risk_score > 0.6:
        burnout_risk = 'high'
        recommendations.append("Consider immediate workload reduction and time off")
    elif risk_score > 0.3:
        burnout_risk = 'medium'
        recommendations.append("Monitor workload and adjust as needed")
    else:
        burnout_risk = 'low'
        if not recommendations:
            recommendations.append("Workload appears healthy")
    
    return BurnoutAnalysis(
        user_id=workload.user_id,
        burnout_risk=burnout_risk,
        risk_score=risk_score,
        indicators=indicators if indicators else ["No burnout indicators detected"],
        recommendations=recommendations
    )
```

## Frontend Integration

### User Dashboard Component
```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [tickets, setTickets] = useState([]);
  const [stats, setStats] = useState({});
  const [loading, setLoading] = useState(true);

  const API_URL = process.env.REACT_APP_API_URL;
  
  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      const [tasksRes, ticketsRes, statsRes] = await Promise.all([
        axios.get(`${API_URL}/api/tasks`, config),
        axios.get(`${API_URL}/api/tickets/my-tickets`, config),
        axios.get(`${API_URL}/api/users/stats`, config)
      ]);

      // Organize tasks by status
      const tasksByStatus = {
        todo: tasksRes.data.data.filter(t => t.status === 'todo'),
        inProgress: tasksRes.data.data.filter(t => t.status === 'in_progress'),
        done: tasksRes.data.data.filter(t => t.status === 'done')
      };

      setTasks(tasksByStatus);
      setTickets(ticketsRes.data.data);
      setStats(statsRes.data.data);
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
      
      fetchDashboardData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const KanbanColumn = ({ title, tasks, status }) => (
    <div className="kanban-column">
      <h3>{title} ({tasks.length})</h3>
      <div className="task-list">
        {tasks.map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <div className="task-meta">
              <span className={`priority-${task.priority}`}>
                {task.priority}
              </span>
              {task.dueDate && (
                <span>Due: {new Date(task.dueDate).toLocaleDateString()}</span>
              )}
            </div>
            <div className="task-actions">
              {status !== 'done' && (
                <button onClick={() => updateTaskStatus(
                  task._id, 
                  status === 'todo' ? 'in_progress' : 'done'
                )}>
                  {status === 'todo' ? 'Start' : 'Complete'}
                </button>
              )}
            </div>
          </div>
        ))}
      </div>
    </div>
  );

  if (loading) return <div>Loading...</div>;

  return (
    <div className="user-dashboard">
      <div className="dashboard-header">
        <h1>My Dashboard</h1>
        <div className="stats-cards">
          <div className="stat-card">
            <h3>{stats.totalTasks || 0}</h3>
            <p>Total Tasks</p>
          </div>
          <div className="stat-card">
            <h3>{stats.completedTasks || 0}</h3>
            <p>Completed</p>
          </div>
          <div className="stat-card">
            <h3>{stats.openTickets || 0}</h3>
            <p>Open Tickets</p>
          </div>
        </div>
      </div>

      <div className="kanban-board">
        <KanbanColumn title="To Do" tasks={tasks.todo} status="todo" />
        <KanbanColumn title="In Progress" tasks={tasks.inProgress} status="in_progress" />
        <KanbanColumn title="Done" tasks={tasks.done} status="done" />
      </div>

      <div className="recent-tickets">
        <h2>Recent Tickets</h2>
        <div className="ticket-list">
          {tickets.slice(0, 5).map(ticket => (
            <div key={ticket._id} className="ticket-item">
              <h4>{ticket.title}</h4>
              <span className={`status-${ticket.status}`}>{ticket.status}</span>
              <span className={`priority-${ticket.priority}`}>{ticket.priority}</span>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component
```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({});
  const [riskUsers, setRiskUsers] = useState([]);
  const [burnoutAlerts, setBurnoutAlerts] = useState([]);
  
  const API_URL = process.env.REACT_APP_API_URL;
  const ML_API_URL = process.env.REACT_APP_ML_API_URL;

  useEffect(() => {
    fetchAnalytics();
    fetchRiskDetection();
    fetchBurnoutAnalysis();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(`${API_URL}/api/admin/analytics`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      setAnalytics(response.data.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const fetchRiskDetection = async () => {
    try {
      const token = localStorage.getItem('token');
      const usersRes = await axios.get(`${API_URL}/api/admin/users`, {
        headers: { Authorization: `Bearer ${token}` }
      });

      // Analyze each user for risk
      const riskAnalysis = await Promise.all(
        usersRes.data.data.map(async (user) => {
          try {
            const activityRes = await axios.get(
              `${API_URL}/api/admin/users/${user._id}/activity`,
              { headers: { Authorization: `Bearer ${token}` } }
            );

            const riskRes = await axios.post(
              `${ML_API_URL}/detect-risk`,
              activityRes.data.data
            );

            return {
              user: user.name,
              userId: user._id,
              ...riskRes.data
            };
          } catch (error) {
            return null;
          }
        })
      );

      const highRiskUsers = riskAnalysis
        .filter(r => r && r.risk_level === 'high')
        .sort((a, b) => b.score - a.score);

      setRiskUsers(highRiskUsers);
    } catch (error) {
      console.error('Error fetching risk detection:', error);
    }
  };

  const fetchBurnoutAnalysis = async () => {
    try {
      const token = localStorage.getItem('token');
      const usersRes = await axios.get(`${API_URL}/api/admin/users`, {
        headers: { Authorization: `Bearer ${token}` }
      });

      const burnoutAnalysis = await Promise.all(
        usersRes.data.data.map(async (user) => {
          try {
            const workloadRes = await axios.get(
              `${API_URL}/api/admin/users/${user._id}/workload`,
              { headers: { Authorization: `Bearer ${token}` } }
            );

            const burnoutRes = await axios.post(
              `${ML_API_URL}/detect-burnout`,
              workloadRes.data.data
            );

            return {
              user: user.name,
              userId: user._id,
              ...burnoutRes.data
            };
          } catch (error) {
            return null;
          }
        })
      );

      const burnoutAlerts = burnoutAnalysis
        .filter(b => b && ['high', 'medium'].includes(b.burnout_risk))
        .sort((a, b) => b.risk_score - a.risk_score);

      setBurnoutAlerts(burnoutAlerts);
    } catch (error) {
      console.error('Error fetching burnout analysis:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Analytics Dashboard</h1>

      <div className="analytics-overview">
        <div className="stat-card">
          <h3>{analytics.totalUsers || 0}</h3>
          <p>Total Users</p>
        </div>
        <div className="stat-card">
          <h3>{analytics.activeTasks || 0}</h3>
          <p>Active Tasks</p>
        </div>
        <div className="stat-card">
          <h3>{analytics.openTickets || 0}</h3>
          <p>Open Tickets</p>
        </div>
        <div className="stat-card">
          <h3>{analytics.completionRate || 0}%</h3>
          <p>Completion Rate</p>
        </div>
      </div>

      <div className="alerts-section">
        <div className="risk-alerts">
          <h2>High Risk Users</h2>
          {riskUsers.length === 0 ? (
            <p>No high-risk users detected</p>
          ) : (
            <div className="alert-list">
              {riskUsers.map((risk) => (
                <div key={risk.userId} className="alert-card risk-alert">
                  <h3>{risk.user}</h3>
                  <div className="risk-score">
                    Risk Score: {(risk.score * 100).toFixed(1)}%
                  </div>
                  <div className="risk-factors">
                    <strong>Factors:</strong>
                    <ul>
                      {Object.entries(risk.factors).map(([key, value]) => (
                        value > 0 && (
                          <li key={key}>
                            {key.replace(/_/g, ' ')}: {(value * 100).toFixed(0)}%
                          </li>
                        )
                      ))}
                    </ul>
                  </div>
                  <div className="recommendations">
                    <strong>Recommendations:</strong>
                    <ul>
                      {risk.recommendations.map((rec, idx) => (
                        <li key={idx}>{rec}</li>
                      ))}
                    </ul>
                  </div>
                </div>
              ))}
            </div>
          )}
        </div>

        <div className="burnout-alerts">
          <h2>Burnout Risk</h2>
          {burnoutAlerts.length === 0 ? (
            <p>No burnout risks detected</p>
          ) : (
            <div className="alert-list
