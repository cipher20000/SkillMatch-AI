# AI-Powered Resume Screener - Features

## Overview
A full-stack application that uses AI to analyze and rank resumes against job descriptions, providing intelligent candidate screening with skill matching and similarity scoring.

## Tech Stack

### Frontend
- **React** with TypeScript
- **Tailwind CSS** for styling
- **Lucide React** for icons
- **Vite** for build tooling

### Backend
- **Supabase Edge Functions** (Deno runtime)
- Custom embedding generation and similarity calculation
- RESTful API endpoints

### Database
- **PostgreSQL** (via Supabase)
- **pgvector** extension for vector embeddings
- Row Level Security (RLS) enabled

### ML/AI
- Custom text embedding generation
- Cosine similarity calculation for resume matching
- Skill extraction using keyword matching
- Text processing and analysis

## Key Features

### 1. Job Description Management
- Upload job descriptions with title and full text
- Automatic skill extraction from job descriptions
- Secure storage with user authentication

### 2. Resume Processing
- **Multi-file upload** support for PDF, DOCX, and TXT files
- Text extraction from various document formats
- Candidate name extraction
- Skills identification and matching

### 3. AI-Powered Analysis
- **Vector embeddings** generation for semantic similarity
- **Cosine similarity scoring** between resumes and job descriptions
- **Skill matching** with percentage calculations
- Automatic ranking based on match quality

### 4. Interactive Dashboard
- **Ranked candidate list** with visual indicators
- **Top performers** highlighted with medal icons
- **Match percentage** badges with color coding
- **Skill visualization** with horizontal bar charts
- **Summary statistics**:
  - Total candidates analyzed
  - Average match percentage
  - Top match score

### 5. Visual Skill Analysis
- Bar chart showing skill match distribution
- Percentage of candidates with each skill
- Top 10 most matched skills displayed
- Color-coded progress bars

### 6. Demo Mode
- Pre-loaded sample resumes for quick testing
- Sample job description included
- One-click demo data loading

## API Endpoints

### POST /functions/v1/analyze-resumes
Analyzes multiple resumes against a job description.

**Request Body:**
```json
{
  "jobDescriptionId": "uuid",
  "resumeTexts": [
    {
      "fileName": "string",
      "text": "string",
      "candidateName": "string (optional)"
    }
  ]
}
```

**Response:**
```json
{
  "success": true,
  "results": [
    {
      "resumeId": "uuid",
      "candidateName": "string",
      "fileName": "string",
      "similarityScore": 0.85,
      "matchPercentage": 75.5,
      "skillMatchCount": 8,
      "matchedSkills": ["React", "TypeScript", ...]
    }
  ]
}
```

## Database Schema

### Tables
1. **job_descriptions** - Stores job postings with requirements
2. **resumes** - Stores candidate resumes and extracted information
3. **resume_scores** - Stores analysis results and match scores

### Security
- All tables protected with Row Level Security (RLS)
- Anonymous authentication supported
- Users can only access their own data

## How It Works

1. **Job Description Input**
   - User enters job title and full description
   - System extracts required skills automatically
   - Generates vector embedding for semantic analysis

2. **Resume Upload**
   - Multiple files can be uploaded simultaneously
   - Text extracted from various formats
   - Skills identified using keyword matching

3. **AI Analysis**
   - Vector embeddings generated for each resume
   - Cosine similarity calculated vs job description
   - Skills matched against requirements
   - Combined score calculated (similarity + skill match)

4. **Results Display**
   - Candidates ranked by match percentage
   - Visual skill breakdown provided
   - Top performers highlighted
   - Detailed skill matching shown per candidate

## Use Cases

- **Recruitment Teams**: Quickly screen large volumes of resumes
- **HR Departments**: Identify top candidates efficiently
- **Hiring Managers**: Compare candidates objectively
- **Job Seekers**: Test resume against job descriptions

## Performance

- Handles multiple resume processing in parallel
- Real-time analysis with progress indicators
- Optimized database queries with indexes
- Efficient vector similarity calculations
