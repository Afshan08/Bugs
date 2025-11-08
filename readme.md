# Python Debugging Examples

This document contains code snippets with moderate-difficulty bugs from various Python projects. These are designed to showcase common issues that aren't immediately obvious but can cause runtime errors or unexpected behavior. Each example includes the buggy code, a brief description of the issue, and hints for debugging.

## Example 1: Incorrect User Authorization Check in Django Todo App

**Buggy Code Snippet:**

```python
@csrf_exempt
@login_required  
def add_task(request):
    if request.method == "POST":
        data = json.loads(request.body)
        user_id = request.user.id
        title = data.get('title')
        description = data.get('description')
        print(f"title\n{title}, \n\ndescription:\n{description}\n\n")
        try: 
            if user_id != request.user.id:
                return JsonResponse({"Error": "You are not authorized to add tasks for this user"}, status=403)
            elif not title or not user_id:
                return JsonResponse({"Error": "Incomplete Information"}, status=400)
            task = TodoList.objects.create(
                user=get_object_or_404(Users, id=user_id),
                title=title,
                description=description,
            )
        except Users.DoesNotExist:
            return JsonResponse({"Error": "User does not exist"}, status=404)
        
        return JsonResponse({'message': 'Task created successfully', 'task_id': task.id})
    else:
        return JsonResponse({"Error": "Bad request"}, status=400)
```

**Issue Description:**  
The authorization check `if user_id != request.user.id:` will never trigger because `user_id` is explicitly set to `request.user.id` right before the check. This makes the condition redundant and potentially confusing, but it doesn't cause an error—it's a logic flaw that could mislead developers into thinking there's actual authorization logic.

**Debugging Hints:**  
- Review variable assignments before conditional checks.  
- Test with different user scenarios to see if the check ever fails.  
- Consider if this is a copy-paste error from another function.

## Example 2: Database Table Creation Timing Issue in Flask App

**Buggy Code Snippet:**

```python
@app.route('/signup', methods=['POST', 'GET'])
def signup():
    if request.method == 'POST':
        email = request.form.get('email')
        password = request.form.get('password')
        hashed_password = generate_password_hash(password, salt_length=8)
        user = User.query.filter_by(email=email).first()
        if not user:
            new_user = User(
                email=email,
                password=hashed_password,
            )
            try:
                db.session.add(new_user)
                db.session.commit()
                login_user(new_user)
            except Exception as e:
                print(f"There was an error {e}")

    return render_template('signup.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        email = request.form.get('email')
        password = request.form.get('password')
        user = User.query.filter_by(email=email).first()
        if not user:
            return ("Sorry but you are not signed up please consider doing so. Here is the link <br>"
                    "<a link='{{url_for('signup')}}' Signup </a>")
        if user:
            if check_password_hash(user.password, password):
                login_user(user)
                return "Congratulations the login was successful. "
    return render_template('login.html')

if __name__ == "__main__":
    app.run(debug=True)
    with app.app_context():
        db.create_all()
```

**Issue Description:**  
The `db.create_all()` is called after `app.run(debug=True)`, which means the database tables are only created when the app is stopped (if ever), not when it starts. This can lead to SQLAlchemy errors when trying to query or commit to non-existent tables during runtime.

**Debugging Hints:**  
- Check the order of operations in the `if __name__ == "__main__":` block.  
- Run the app and attempt a signup/login to see if tables exist.  
- Look for SQLAlchemy exceptions related to missing tables.

## Example 3: Potential IndexError in Dashboard Statistics Calculation

**Buggy Code Snippet:**

```python
def mode_result(user_id):
    mode_result = (
        db.session.query(UserData.question_id, func.count(UserData.question_id).label("frequency"))
        .filter_by(user_id=user_id, is_correct=False)  # Filter for specific user_id and incorrect answers
        .group_by(UserData.question_id)  # Group by question_id
        .order_by(func.count(UserData.question_id).desc())  # Sort by highest count
        .limit(1)  # Get the most frequent value
        .first()
        )

    return mode_result[0] if mode_result else None

# In dashboard route:
@login_required
def dashboard():
    # ... other code ...
    the_id_for_wrong_question = mode_result(current_user.id)
    repeated_mistake = db.session.query(Questions.question).filter_by(id=the_id_for_wrong_question).scalar()
    answer_to_the_question = db.session.query(Questions.answer).filter_by(id=the_id_for_wrong_question).scalar()
    # ... rest of function ...
```

**Issue Description:**  
If `mode_result` returns `None` (when the user has no incorrect answers), then `the_id_for_wrong_question` is `None`, and the subsequent queries will fail or return `None`, but if the code assumes it's an integer, it might cause issues downstream. However, the bug is subtle: if there are results but the query returns a tuple, accessing `[0]` is fine, but if no data, it returns `None`, and the scalar queries might not handle `None` gracefully.

**Debugging Hints:**  
- Test with a user who has no incorrect attempts.  
- Add print statements to check the value of `the_id_for_wrong_question`.  
- Ensure queries handle `None` inputs properly.
