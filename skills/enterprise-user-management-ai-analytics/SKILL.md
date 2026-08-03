---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management with AI analytics"
  - "integrate AI-based user management system"
  - "build admin dashboard with user task tracking"
  - "implement AI ticket classification and routing"
  - "create user management with burnout detection"
  - "add predictive analytics to user management"
  - "configure AI-powered support ticket system"
  - "deploy enterprise user management with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform that combines user/task administration with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights. Built with React frontend, Node.js/Spring Boot backend, FastAPI ML service, and MongoDB database.

## What This Project Does

This system provides a centralized platform for managing organizational users, tasks, and support tickets with intelligent automation:

- **User Management**: Secure authentication, role-based access control, personal dashboards
- **Task Tracking**: Kanban boards (To Do → In Progress → Done), time tracking, progress monitoring
- **Support System**: AI-classified ticket routing, smart ticket management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Tools**: User CRUD operations, audit logs, organization analytics, security alerts

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+ and pip
- MongoDB instance (local or cloud)

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
```

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Frontend runs at http://localhost:3000
```

## Key API Endpoints

### Authentication (Backend)

```javascript
// POST /api/auth/register - Register new user
const registerUser = async (userData) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/register`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login - User login
const loginUser = async (credentials) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Backend)

```javascript
// GET /api/users - Get all users (admin only)
const getAllUsers = async (token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/users`, {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, userData, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(userData)
  });
  return response.json();
};

// DELETE /api/users/:id - Delete user (admin only)
const deleteUser = async (userId, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management (Backend)

```javascript
// POST /api/tasks - Create new task
const createTask = async (taskData, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'inprogress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// GET /api/tasks/my-tasks - Get current user's tasks
const getMyTasks = async (token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/my-tasks`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, newStatus, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};
```

### Support Tickets (Backend)

```javascript
// POST /api/tickets - Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tickets`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets
const getTickets = async (token, filters = {}) => {
  const queryParams = new URLSearchParams(filters);
  const response = await fetch(
    `${process.env.REACT_APP_API_URL}/tickets?${queryParams}`,
    {
      headers: {
        'Authorization': `Bearer ${token}`
      }
    }
  );
  return response.json();
};
```

### AI/ML Service Endpoints

```javascript
// POST /ai/classify-ticket - AI ticket classification
const classifyTicket = async (ticketText) => {
  const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/ai/classify-ticket`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: "Support request"
    })
  });
  return response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.87 }
};

// POST /ai/risk-prediction - Predict user risk score
const predictUserRisk = async (userId, token) => {
  const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/ai/risk-prediction`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      recent_activities: [], // Activity log data
      task_completion_rate: 0.75,
      avg_response_time: 24
    })
  });
  return response.json();
  // Returns: { risk_score: 0.35, risk_level: 'low', factors: [...] }
};

// POST /ai/burnout-detection - Detect user burnout
const detectBurnout = async (userId, token) => {
  const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/ai/burnout-detection`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      workload_hours: 45,
      tasks_completed: 12,
      tasks_pending: 8,
      stress_indicators: []
    })
  });
  return response.json();
  // Returns: { burnout_risk: 'medium', score: 0.62, recommendations: [...] }
};

// POST /ai/anomaly-detection - Detect anomalous behavior
const detectAnomaly = async (activityData, token) => {
  const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/ai/anomaly-detection`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: activityData.userId,
      login_time: activityData.loginTime,
      location: activityData.location,
      device: activityData.device,
      activity_pattern: activityData.pattern
    })
  });
  return response.json();
  // Returns: { is_anomaly: false, anomaly_score: 0.12, details: {...} }
};

// POST /ai/project-insights - Get predictive project insights
const getProjectInsights = async (projectData, token) => {
  const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/ai/project-insights`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      project_id: projectData.projectId,
      tasks_total: projectData.totalTasks,
      tasks_completed: projectData.completedTasks,
      deadline: projectData.deadline,
      team_size: projectData.teamSize
    })
  });
  return response.json();
  // Returns: { delay_probability: 0.23, estimated_completion: '2026-05-15', risks: [...] }
};
```

## Common Patterns

### React Component with Authentication

```javascript
import React, { useState, useEffect } from 'react';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchUserTasks();
  }, []);

  const fetchUserTasks = async () => {
    try {
      const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/my-tasks`, {
        headers: {
          'Authorization': `Bearer ${token}`
        }
      });
      const data = await response.json();
      setTasks(data.tasks);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
        method: 'PATCH',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ status: newStatus })
      });
      fetchUserTasks(); // Refresh tasks
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="dashboard">
      <h1>My Tasks</h1>
      {loading ? (
        <p>Loading...</p>
      ) : (
        <div className="kanban-board">
          {['todo', 'inprogress', 'done'].map(status => (
            <div key={status} className="kanban-column">
              <h2>{status.toUpperCase()}</h2>
              {tasks.filter(t => t.status === status).map(task => (
                <div key={task._id} className="task-card">
                  <h3>{task.title}</h3>
                  <p>{task.description}</p>
                  <button onClick={() => updateTaskStatus(task._id, getNextStatus(status))}>
                    Move to {getNextStatus(status)}
                  </button>
                </div>
              ))}
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

const getNextStatus = (currentStatus) => {
  const statusFlow = { todo: 'inprogress', inprogress: 'done', done: 'todo' };
  return statusFlow[currentStatus];
};

export default UserDashboard;
```

### Admin User Management Component

```javascript
import React, { useState, useEffect } from 'react';

const AdminUserManagement = () => {
  const [users, setUsers] = useState([]);
  const [newUser, setNewUser] = useState({ name: '', email: '', role: 'user' });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchUsers();
  }, []);

  const fetchUsers = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/users`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    setUsers(data.users);
  };

  const handleCreateUser = async (e) => {
    e.preventDefault();
    try {
      await fetch(`${process.env.REACT_APP_API_URL}/auth/register`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(newUser)
      });
      setNewUser({ name: '', email: '', role: 'user' });
      fetchUsers();
    } catch (error) {
      console.error('Error creating user:', error);
    }
  };

  const handleDeleteUser = async (userId) => {
    if (window.confirm('Are you sure you want to delete this user?')) {
      try {
        await fetch(`${process.env.REACT_APP_API_URL}/users/${userId}`, {
          method: 'DELETE',
          headers: { 'Authorization': `Bearer ${token}` }
        });
        fetchUsers();
      } catch (error) {
        console.error('Error deleting user:', error);
      }
    }
  };

  return (
    <div className="admin-panel">
      <h1>User Management</h1>
      
      <form onSubmit={handleCreateUser}>
        <input
          type="text"
          placeholder="Name"
          value={newUser.name}
          onChange={(e) => setNewUser({...newUser, name: e.target.value})}
          required
        />
        <input
          type="email"
          placeholder="Email"
          value={newUser.email}
          onChange={(e) => setNewUser({...newUser, email: e.target.value})}
          required
        />
        <select
          value={newUser.role}
          onChange={(e) => setNewUser({...newUser, role: e.target.value})}
        >
          <option value="user">User</option>
          <option value="admin">Admin</option>
        </select>
        <button type="submit">Add User</button>
      </form>

      <table>
        <thead>
          <tr>
            <th>Name</th>
            <th>Email</th>
            <th>Role</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {users.map(user => (
            <tr key={user._id}>
              <td>{user.name}</td>
              <td>{user.email}</td>
              <td>{user.role}</td>
              <td>
                <button onClick={() => handleDeleteUser(user._id)}>Delete</button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};

export default AdminUserManagement;
```

### AI-Powered Ticket Creation

```javascript
import React, { useState } from 'react';

const CreateTicket = () => {
  const [ticket, setTicket] = useState({
    subject: '',
    description: '',
    priority: 'medium',
    category: ''
  });
  const [aiSuggestions, setAiSuggestions] = useState(null);
  const token = localStorage.getItem('token');

  const handleAIClassify = async () => {
    try {
      const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/ai/classify-ticket`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          text: ticket.description,
          subject: ticket.subject
        })
      });
      const data = await response.json();
      setAiSuggestions(data);
      setTicket({
        ...ticket,
        category: data.category,
        priority: data.priority
      });
    } catch (error) {
      console.error('AI classification error:', error);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await fetch(`${process.env.REACT_APP_API_URL}/tickets`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(ticket)
      });
      alert('Ticket created successfully!');
      setTicket({ subject: '', description: '', priority: 'medium', category: '' });
      setAiSuggestions(null);
    } catch (error) {
      console.error('Error creating ticket:', error);
    }
  };

  return (
    <div className="create-ticket">
      <h2>Create Support Ticket</h2>
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          placeholder="Subject"
          value={ticket.subject}
          onChange={(e) => setTicket({...ticket, subject: e.target.value})}
          required
        />
        <textarea
          placeholder="Description"
          value={ticket.description}
          onChange={(e) => setTicket({...ticket, description: e.target.value})}
          required
        />
        <button type="button" onClick={handleAIClassify}>
          AI Classify
        </button>
        
        {aiSuggestions && (
          <div className="ai-suggestions">
            <p>AI Suggestions (Confidence: {(aiSuggestions.confidence * 100).toFixed(0)}%)</p>
            <p>Category: {aiSuggestions.category}</p>
            <p>Priority: {aiSuggestions.priority}</p>
          </div>
        )}

        <select
          value={ticket.priority}
          onChange={(e) => setTicket({...ticket, priority: e.target.value})}
        >
          <option value="low">Low</option>
          <option value="medium">Medium</option>
          <option value="high">High</option>
        </select>

        <input
          type="text"
          placeholder="Category"
          value={ticket.category}
          onChange={(e) => setTicket({...ticket, category: e.target.value})}
        />

        <button type="submit">Submit Ticket</button>
      </form>
    </div>
  );
};

export default CreateTicket;
```

### Analytics Dashboard with AI Insights

```javascript
import React, { useState, useEffect } from 'react';

const AnalyticsDashboard = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeTasks: 0,
    burnoutAlerts: [],
    riskUsers: [],
    anomalies: []
  });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch basic stats
      const statsResponse = await fetch(`${process.env.REACT_APP_API_URL}/analytics/stats`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const stats = await statsResponse.json();

      // Fetch AI insights
      const insightsResponse = await fetch(`${process.env.REACT_APP_ML_API_URL}/ai/insights`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const insights = await insightsResponse.json();

      setAnalytics({
        totalUsers: stats.totalUsers,
        activeTasks: stats.activeTasks,
        burnoutAlerts: insights.burnout_alerts,
        riskUsers: insights.high_risk_users,
        anomalies: insights.recent_anomalies
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="analytics-dashboard">
      <h1>Organization Analytics</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p>{analytics.totalUsers}</p>
        </div>
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p>{analytics.activeTasks}</p>
        </div>
      </div>

      <div className="ai-insights">
        <h2>AI Insights</h2>
        
        <div className="burnout-alerts">
          <h3>Burnout Alerts</h3>
          {analytics.burnoutAlerts.length === 0 ? (
            <p>No burnout alerts</p>
          ) : (
            <ul>
              {analytics.burnoutAlerts.map((alert, idx) => (
                <li key={idx}>
                  {alert.user_name} - Risk: {alert.risk_level} (Score: {alert.score})
                </li>
              ))}
            </ul>
          )}
        </div>

        <div className="risk-users">
          <h3>High Risk Users</h3>
          {analytics.riskUsers.length === 0 ? (
            <p>No high-risk users detected</p>
          ) : (
            <ul>
              {analytics.riskUsers.map((user, idx) => (
                <li key={idx}>
                  {user.name} - Risk Score: {user.risk_score}
                </li>
              ))}
            </ul>
          )}
        </div>

        <div className="anomalies">
          <h3>Recent Anomalies</h3>
          {analytics.anomalies.length === 0 ? (
            <p>No anomalies detected</p>
          ) : (
            <ul>
              {analytics.anomalies.map((anomaly, idx) => (
                <li key={idx}>
                  {anomaly.type} - User: {anomaly.user_name} at {new Date(anomaly.timestamp).toLocaleString()}
                </li>
              ))}
            </ul>
          )}
        </div>
      </div>
    </div>
  );
};

export default AnalyticsDashboard;
```

## Configuration

### Backend Environment Variables

```env
# Server Configuration
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise-ums

# Authentication
JWT_SECRET=your_secure_jwt_secret_key
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional for notifications)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password

# Rate Limiting
RATE_LIMIT_WINDOW=15
RATE_LIMIT_MAX_REQUESTS=100
```

### ML Service Configuration

```env
# ML Service Configuration
MONGODB_URI=mongodb://localhost:27017/enterprise-ums

# Model Storage
MODEL_PATH=./models
CHECKPOINT_INTERVAL=3600

# Logging
LOG_LEVEL=INFO
LOG_FILE=./logs/ml_service.log

# AI Thresholds
RISK_THRESHOLD=0.7
ANOMALY_THRESHOLD=0.8
BURNOUT_THRESHOLD=0.6

# Model Retraining
AUTO_RETRAIN=true
RETRAIN_INTERVAL=86400
```

### Frontend Environment Variables

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_VERSION=1.0.0
```

## Database Schema Examples

### User Schema (MongoDB)

```javascript
// User model structure
{
  _id: ObjectId,
  name: String,
  email: String (unique),
  password: String (hashed),
  role: String, // 'user' or 'admin'
  createdAt: Date,
  updatedAt: Date,
  isActive: Boolean,
  lastLogin: Date,
  profile: {
    department: String,
    position: String,
    phoneNumber: String
  }
}
```

### Task Schema

```javascript
// Task model structure
{
  _id: ObjectId,
  title: String,
  description: String,
  assignedTo: ObjectId (ref: User),
  createdBy: ObjectId (ref: User),
  status: String, // 'todo', 'inprogress', 'done'
  priority: String, // 'low', 'medium', 'high'
  dueDate: Date,
  completedAt: Date,
  timeTracked: Number, // seconds
  createdAt: Date,
  updatedAt: Date,
  tags: [String]
}
```

### Ticket Schema

```javascript
// Support ticket model structure
{
  _id: ObjectId,
  subject: String,
  description: String,
  createdBy: ObjectId (ref: User),
  assignedTo: ObjectId (ref: User),
  status: String, // 'open', 'in_progress', 'resolved', 'closed'
  priority: String, // 'low', 'medium', 'high', 'urgent'
  category: String,
  aiClassification: {
    category: String,
    priority: String,
    confidence: Number
  },
  createdAt: Date,
  updatedAt: Date,
  resolvedAt: Date,
  comments: [{
    user: ObjectId (ref: User),
    text: String,
    timestamp: Date
  }]
}
```

## Troubleshooting

### Common Issues

**Issue: JWT Token Expired**
```javascript
// Implement token refresh logic
const refreshToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/refresh`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken })
  });
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data.token;
};
```

**Issue: CORS Errors in Development**
```javascript
// Backend: Add CORS middleware (Express)
const cors = require('cors');
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

**Issue: ML Service Connection Failed**
```javascript
// Add error handling and fallback
const getAIInsights = async (data, token) => {
  try {
    const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/ai/insights`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(data),
      timeout: 5000
    });
    return await response.json();
  } catch (error) {
    console.error('ML service unavailable:', error);
    return { error: 'AI service temporarily unavailable', fallback: true };
  }
};
```

**Issue: MongoDB Connection Timeout**
```javascript
// Backend: Configure MongoDB connection options
const mongoose = require('mongoose');
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
  maxPoolSize: 10
});
```

**Issue: High Memory Usage in ML Service**
```python
# ml-service: Implement model caching and cleanup
import gc
from functools import lru_cache

@lru_cache(maxsize=10)
def load_model(model_name):
    # Load model with caching
    return pickle.load(open(f"./models/{model_name}.pkl", "rb"))

# Periodic cleanup
def cleanup_memory():
    gc.collect()
```

**Issue: Real-time Updates Not Working**
```javascript
// Implement polling or WebSocket for real-time updates
const useTaskUpd
