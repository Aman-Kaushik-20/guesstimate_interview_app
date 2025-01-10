# Guesstimate Interview Assistant 🤖

A Streamlit-based application that conducts automated guesstimate interviews using Claude AI, providing real-time feedback and evaluation.

## Features

### Core Functionality
- Interactive AI-powered interview experience
- Real-time conversation with dynamic responses
- Automated evaluation and scoring system
- PDF transcript generation and download
- Token usage tracking and cost management
- Google Sheets integration for feedback collection

### Interview Components
- Problem generation from predefined statements
- Clarifying questions support
- Real-time feedback on approach
- Structured evaluation across multiple criteria
- Detailed performance analysis

## Key Code Components

### 1. GuesstimateChatbot Class

```python
class GuesstimateChatbot:
    def __init__(self, api_key: str):
        self.client = anthropic.Anthropic(api_key=api_key)
        self.interview_data = self.load_interview_data("interview_with_context.json")
        self.conversation_history = []
        self.evaluation_criteria = [
            "Structured Approach",
            "Logical Assumptions",
            "Segmentation Strategy",
            "Mathematical Accuracy",
            "Context Awareness"
        ]
        # Token tracking initialization
        self.token_stats = {
            'input_tokens': 0,
            'output_tokens': 0,
            'input_cost': 0,
            'output_cost': 0,
            'total_cost': 0
        }
        self.cost_rates = {
            'input': 0.000003,      # $3/MTok
            'output': 0.000015,     # $15/MTok
            'cache_write': 0.00000375,  # $3.75/MTok
            'cache_read': 0.0000003    # $0.30/MTok
        }
        self.max_cost_limit = 0.25  # $0.25 limit
        self.system_prompt = self.create_system_prompt()
```

### 2. Interview Management Functions

```python
def conduct_interview(self, candidate_response: str) -> str:
    """Conduct one turn of the guesstimate interview with token tracking"""
    # Estimate input tokens (rough estimate: 4 chars = 1 token)
    estimated_input_tokens = len(candidate_response) // 4
    
    # Check if this would exceed our cost limit
    if self.would_exceed_cost_limit(estimated_input_tokens):
        return "COST_LIMIT_EXCEEDED"
    
    # Add candidate's response to history
    self.conversation_history.append({
        "role": "user",
        "content": candidate_response
    })
    
    # Create messages for API call
    messages = []
    if self.current_problem:
        messages.append({
            "role": "assistant",
            "content": f"Current estimation problem: {self.current_problem}"
        })
    
    messages.extend([
        {"role": "user" if msg["role"] == "user" else "assistant", "content": msg["content"]}
        for msg in self.conversation_history
    ])
    
    # Get response from Claude
    response = self.client.messages.create(
        model="claude-3-5-sonnet-20241022",
        messages=messages,
        system=self.system_prompt,
        max_tokens=1024,
        temperature=0.7
    )
    
    # Update token statistics
    self.update_token_stats(response)
    
    # Add Claude's response to history
    self.conversation_history.append({
        "role": "assistant",
        "content": response.content[0].text
    })
    
    return response.content[0].text


    def create_system_prompt(self) -> str:
        """Create system prompt for guesstimate interviews"""
        prompt = """You are an expert interviewer specializing in guesstimate questions. Your role is to:

Filters that can be used in problem for approaching problem :

   - Demographics: Age, gender, income level 
   - Geography: City, region, urban vs. rural 
   - Behavior: Usage frequency, product preference, online vs. offline activity 
   - Socioeconomic factors: Education level, occupation
   - Population segmentation
   - Regional variations
   - Income levels
   - Behavioral patterns
   - Seasonal factors


Follow these guidelines:
1. Start with a clear problem statement.
2. Let Candidate Ask clarifying questions about their methodology. You will not give suggestions on clarifying questions , let them ask their own question.
3. Give Only Relevant , Shorter and required answers to clarifying Questions, nothing like -That's a good clarifying question , just give answer, Keep Answers/Clarifying questions around India Only wherver region/ Place not mentioned.
4. Once Clarifying Questions are done, candidate will start his/her approach.
5. Ask or Challenge with Relevant Questions for their assumptions,  calculations, reasoning and filters for segementation they are using,  If necessary.
6. Don't Suggest Next Filters of calculations, Let candidate do their own thing.
7. Once Candidate is done, You will Give review of their approach - the mistakes in assumptions , the filters that they missed and Improvements that could be done.


Example interview patterns from real interviews -:
"""
        # Add example exchanges from the training data
        for interview in self.interview_data['interviews'][:2]:
            prompt += f"\nExample for {interview['topic']}:\n"
            for exchange in interview['exchanges'][:20]:
                prompt += f"{exchange['role'].title()}: {exchange['content']}\n"
        
        return prompt
    
    def select_problem(self) -> str:
        """Select a random problem statement or create a new one From following, You don't have to necessarily choose these , you can make on your own also"""
        return random.choice(self.interview_data['problem_statements'])
    
    def start_interview(self) -> str:
        """Start a new interview with a problem statement"""
        self.conversation_history = []
        self.current_problem = self.select_problem()
        return f"Your problem statement is to calculate {self.current_problem}. Please provide your approach to estimate this value."


```

### 3. Evaluation System

```python
def evaluate_candidate(self) -> Dict:
    """Evaluate the candidate's guesstimate approach"""
    evaluation_prompt = """Based on the interview conversation, please evaluate:
        1. 'structure': Did they break down the problem logically. Rate out of 5?
        2. 'assumptions': Were assumptions reasonable and India-specific. Rate out of 5?
        3. 'segmentation': How well did they segment the problem. Rate out of 5?
        4. 'math': Were calculations logical and error-free. Rate out of 5?
        5. 'context': Did they consider context of the problem appropriately. Rate out of 5?
        6. 'filters_missed': (they missed and needed to added in string)
        7. 'key_strengths': (Good points in the approach in string) 
        8. 'areas_for_improvement': (Areas in approach to improve in string)
        Return these in JSON format."""
    
    messages = [
        {
            "role": "user",
            "content": f"Problem: {self.current_problem}\n\nInterview transcript: {json.dumps(self.conversation_history)}\n\n{evaluation_prompt}"
        }
    ]
    
    response = self.client.messages.create(
        model="claude-3-5-sonnet-20241022",
        messages=messages,
        system=self.system_prompt,
        max_tokens=1024,
        temperature=0.3
    )
    
    try:
        evaluation_json = json.loads(response.content[0].text)
        return evaluation_json
    except json.JSONDecodeError as e:
        print(f"Error parsing JSON: {e}")
        return None
```

### 4. Token and Cost Management

```python
def update_token_stats(self, response) -> None:
    """Update token usage and cost statistics"""
    input_tokens = response.usage.input_tokens
    output_tokens = response.usage.output_tokens
    
    self.token_stats['input_tokens'] += input_tokens
    self.token_stats['output_tokens'] += output_tokens
    
    input_cost = input_tokens * 0.000003  # $3 per million tokens
    output_cost = output_tokens * 0.000015  # $15 per million tokens
    
    self.token_stats['input_cost'] += input_cost
    self.token_stats['output_cost'] += output_cost
    self.token_stats['total_cost'] = self.token_stats['input_cost'] + self.token_stats['output_cost']

def would_exceed_cost_limit(self, estimated_input_tokens: int) -> bool:
    """Check if adding more tokens would exceed the cost limit"""
    estimated_cost = (
        (estimated_input_tokens * self.cost_rates["input"]) +
        (estimated_input_tokens * self.cost_rates["cache_write"]) +
        (estimated_input_tokens * 1.5 * self.cost_rates["output"])  # Assume output is 1.5x input
    )
    
    return (self.token_stats['total_cost'] + estimated_cost) > self.max_cost_limit
```

### 5. PDF Generation

```python
def download_interview_transcript(conversation_history: list, evaluation: dict) -> str:
    """Generate a downloadable transcript of the conversation history as a PDF file"""
    pdf = FPDF()
    pdf.set_auto_page_break(auto=True, margin=15)
    pdf.add_page()
    pdf.set_font("Arial", size=12)
    
    # Title
    pdf.set_font("Arial", style="B", size=16)
    pdf.cell(200, 10, txt="Guesstimate Interview Transcript", ln=True, align='C')
    pdf.ln(10)
    
    # Date and Time
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    pdf.set_font("Arial", size=12)
    pdf.cell(0, 10, txt=f"Date and Time: {timestamp}", ln=True)
    pdf.ln(10)
    
    # Conversation History
    pdf.set_font("Arial", style="B", size=12)
    pdf.cell(0, 10, txt="Conversation History:", ln=True)
    pdf.ln(5)
    pdf.set_font("Arial", size=12)
    
    for msg in conversation_history:
        role = "Interviewer" if msg["role"] == "assistant" else "Candidate"
        content = msg['content'].encode('latin1', 'replace').decode('latin1')
        pdf.multi_cell(0, 10, txt=f"{role}: {content}")
        pdf.ln(2)

    # Evaluation Section
    pdf.ln(10)
    pdf.set_font("Arial", style="B", size=12)
    pdf.cell(0, 10, txt="Evaluation Results:", ln=True)
    pdf.ln(5)
    pdf.set_font("Arial", size=12)
    evaluation_text = json.dumps(evaluation, indent=2).encode('latin1', 'replace').decode('latin1')
    pdf.multi_cell(0, 10, txt=evaluation_text)
    
    # Save the PDF
    file_name = f"interview_transcript_{timestamp.replace(':', '-').replace(' ', '_')}.pdf"
    pdf.output(file_name)
    return file_name
```

## Technical Requirements

### Dependencies
```python
streamlit
anthropic
plotly
fpdf
pandas
streamlit_gsheets
datetime
json
```

### API Requirements
- Anthropic API key
- Google Sheets API configuration
- Connection to specified Google Sheets document

### Configuration
- Maximum token limit settings
- Cost tracking parameters
- System prompt customization
- Interview data configuration
