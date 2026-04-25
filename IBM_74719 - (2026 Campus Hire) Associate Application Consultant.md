🧠 STEP 1: Consultant Thinking (Foundation)

This is where most people fail.
If you get this right → you already stand out.

🎯 Your First Scenario

👉 Problem:
A college wants to track student attendance automatically using technology.

❌ What most candidates do:

They jump straight to:

“We use AI… we use camera…”

That’s messy thinking.

✅ What YOU will do (Structured Answer)

Use this exact flow:

1. Understand the Problem (The "Why")
   - **Operational Inefficiency**: Manual registers waste 10-15 minutes per lecture, reducing actual teaching time.
   - **Data Integrity**: "Proxy attendance" is common, leading to inaccurate records and academic dishonesty.
   - **Lack of Real-time Insight**: Administrators have no visibility into student location or campus density during emergencies.
2. Identify Inputs (The "What")
   - **Hardware**: High-definition IP cameras positioned at strategic choke points (main gates, department entries).
   - **Student Master Data**: Existing student database (Name, ID, Photo) and facial embeddings.
   - **Temporal Context**: Classroom timetables and holiday calendars to validate "Expected vs. Actual" attendance.
3. Design Solution (The "How")
   - **Detection & Recognition**: Use an Edge-based AI model to detect faces in real-time without sending raw video to the cloud (Privacy-first).
   - **Verification Logic**: Compare captured frames against the facial vector database using a matching engine.
   - **Automated Logging**: System automatically flags a student "Present" if they are identified at the classroom door within the first 10 minutes of a scheduled slot.
4. Data Handling (The "Storage")
   - **Relational Storage**: Use SQL databases for audit-ready logs (StudentID, LocationID, Timestamp, SessionID).
   - **Dwell Time Tracking**: Record both "In" and "Out" times to distinguish between a student attending a class vs. just walking past it.
   - **Security**: Encrypt facial data and purge transient video data to comply with institutional privacy policies.
5. Output / Value (The "Impact")
   - **Stakeholder Dashboards**: Real-time attendance heatmaps for faculty and individual progress trackers for students.
   - **Proactive Alerts**: Automated SMS/Email triggers to students and mentors when attendance falls below the 75% threshold.
   - **Admin Reports**: Exportable data for statutory compliance, payroll (for staff), and resource optimization (which rooms are underutilized?).
