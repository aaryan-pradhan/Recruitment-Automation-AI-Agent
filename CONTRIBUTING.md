# Contributing

## Attribution

This project's application code — the Django backend, the Gemini integration,
the Google/LinkedIn OAuth flows, the background task pipeline, and the
production-hardening work behind it — was originally designed and built by
[Mohit Gupta](https://github.com/mohitgupta4873). The original repository is
at [mohitgupta4873/Recruitment_Automation_AI_Agent](https://github.com/mohitgupta4873/Recruitment_Automation_AI_Agent).

This fork is maintained with his permission. Contributions here build on that
original work rather than replacing it.

## Making changes

1. Open an issue or discussion before starting significant work, so effort
   isn't duplicated.
2. Follow the existing code style — there's no separate linter/formatter
   configured, so match the conventions already in the file you're editing.
3. Run the test suite before opening a PR:

   ```bash
   python manage.py test
   ```

4. Keep commits scoped to one logical change each, with a commit message that
   explains *why*, not just *what*.
