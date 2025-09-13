from flask import Flask, request, jsonify
import subprocess, tempfile, os, sys

app = Flask(__name__)

# --- Helpers ---
def run_python_code(code):
    try:
        with tempfile.NamedTemporaryFile(delete=False, suffix=".py") as tmp:
            tmp.write(code.encode("utf-8"))
            tmp_path = tmp.name

        result = subprocess.run(
            [sys.executable, tmp_path],
            capture_output=True,
            text=True,
            timeout=5
        )
        os.remove(tmp_path)
        return {"stdout": result.stdout, "stderr": result.stderr}
    except subprocess.TimeoutExpired:
        return {"stdout": "", "stderr": "Execution timed out."}
    except Exception as e:
        return {"stdout": "", "stderr": str(e)}


def analyze_code(code):
    issues = []
    if len(code.strip()) == 0:
        issues.append("Code is empty!")
    if "print(" not in code:
        issues.append("No print statement found — are you showing output?")
    if "eval(" in code:
        issues.append("Avoid using eval() for safety reasons.")
    return issues


def suggest_debugging(stderr):
    if "SyntaxError" in stderr:
        return "Check your syntax — did you forget a colon or parenthesis?"
    if "NameError" in stderr:
        return "You might be using a variable before defining it."
    if "TypeError" in stderr:
        return "Check variable types — are you mixing strings and numbers?"
    if "ZeroDivisionError" in stderr:
        return "Watch out — dividing by zero is not allowed."
    return "Try adding print statements to trace values."


# --- Routes ---
@app.route("/run", methods=["POST"])
def run_code():
    data = request.get_json()
    code = data.get("code", "")
    result = run_python_code(code)
    analysis = analyze_code(code)
    suggestion = suggest_debugging(result["stderr"]) if result["stderr"] else "No issues detected."

    return jsonify({
        "output": result["stdout"],
        "errors": result["stderr"],
        "analysis": analysis,
        "suggestion": suggestion
    })


@app.route("/exercise", methods=["GET"])
def get_exercise():
    # Example static exercise, you can expand with DB later
    return jsonify({
        "id": 1,
        "title": "Print Hello World",
        "instructions": "Write a program that prints 'Hello, World!'",
        "expected_output": "Hello, World!\n"
    })


@app.route("/grade", methods=["POST"])
def grade_code():
    data = request.get_json()
    code = data.get("code", "")
    expected = data.get("expected_output", "")
    result = run_python_code(code)
    passed = result["stdout"] == expected and result["stderr"] == ""
    return jsonify({
        "passed": passed,
        "your_output": result["stdout"],
        "errors": result["stderr"]
    })


if __name__ == "__main__":
    app.run(debug=True)
