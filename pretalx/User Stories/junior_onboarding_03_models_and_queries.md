# Onboarding Stories 21–30: Models & Simple Queries

This module contains introductory stories on database models and query sets to help you learn model fields, query orders, simple table joins, and dynamic properties.

---

## Story 21: Add a property `is_accepted` to the `Submission` model

### Stakeholder Request
*"In multiple backend views, we need to check if a submission's state is currently 'accepted'. Please expose a simple property `is_accepted` that returns a boolean."*

### Junior Walkthrough & Explanation
Django models support standard **Python properties** (`@property`). This allows us to calculate dynamic attributes on database rows on-the-fly without creating a new column in our database schema.

### Detailed Implementation Steps
1.  **Locate Model:** Open `src/pretalx/submission/models/submission.py`.
2.  **Add `@property` method:**
    ```python
    @property
    def is_accepted(self) -> bool:
        return self.state == SubmissionStates.ACCEPTED
    ```

---

## Story 22: Order tracks alphabetically by default in database queries

### Stakeholder Request
*"Whenever tracks are listed in selectors, lists, or public agenda details, they appear in the order they were saved, which looks messy. Let's list them alphabetically by default."*

### Junior Walkthrough & Explanation
Django models define default database query ordering inside the internal class `Meta` using the `ordering` property array.

### Detailed Implementation Steps
1.  **Locate Track Model:** Open `src/pretalx/submission/models/track.py`.
2.  **Edit Ordering list:**
    *   Find the class `Meta` (usually at the bottom of the class):
        ```python
        class Meta:
            # Change or add ordering
            ordering = ("name",)
        ```

---

## Story 23: Add a helper method to check if a user is an organizer

### Stakeholder Request
*"We need a fast, simple method to verify if a logged-in user belongs to the active event's organizing board."*

### Junior Walkthrough & Explanation
We can check event-specific permissions or membership by querying role relationships attached to standard User profiles.

### Detailed Implementation Steps
1.  **Locate User Model:** Open `src/pretalx/person/models/user.py`.
2.  **Add Helper Method:**
    ```python
    def is_organizer(self, event) -> bool:
        # Queries if the user belongs to any team with organizer roles on the event
        return self.teams.filter(event=event, is_reviewer=False).exists()
    ```

---

## Story 24: Expose the number of speakers on the `Submission` model

### Stakeholder Request
*"We want to display the number of speakers assigned to a proposal next to its title. Please add a method to get the total count."*

### Junior Walkthrough & Explanation
Many-to-many relationship rows (like speakers assigned to a talk) can be queried and counted using standard Django reverse manager lookups (`.speakers.count()`).

### Detailed Implementation Steps
1.  **Locate Submission Model:** Open `src/pretalx/submission/models/submission.py`.
2.  **Add speakers count method:**
    ```python
    @property
    def speaker_count(self) -> int:
        return self.speakers.count()
    ```

---

## Story 25: Add a simple boolean field `is_featured` to the `Submission` model

### Stakeholder Request
*"We want to mark specific talks as keynotes or 'featured sessions' so we can highlight them on our public landing page. Please add an 'is_featured' check to submissions."*

### Junior Walkthrough & Explanation
To store new static attributes, we add database fields (e.g. `models.BooleanField`) and execute SQL DDL DML migrations.

### Detailed Implementation Steps
1.  **Locate Submission Model:** Open `src/pretalx/submission/models/submission.py`.
2.  **Add BooleanField:**
    ```python
    is_featured = models.BooleanField(default=False)
    ```
3.  **Generate & Run Migration:**
    ```powershell
    python src/manage.py makemigrations submission
    python src/manage.py migrate
    ```

---

## Story 26: Expose a property `is_cfp_active` on the `Event` model

### Stakeholder Request
*"We need to verify if the Call for Proposals deadline has not passed. Please expose a simple check `is_cfp_active` returning a boolean."*

### Junior Walkthrough & Explanation
We write a simple date logic property comparing timezone-aware timestamps against the current date.

### Detailed Implementation Steps
1.  **Locate Event Model:** Open `src/pretalx/event/models/event.py`.
2.  **Add Date Comparison Method:**
    ```python
    from django.utils import timezone

    @property
    def is_cfp_active(self) -> bool:
        cfp = getattr(self, "cfp", None)
        if not cfp or not cfp.deadline:
            return False
        return cfp.deadline > timezone.now()
    ```

---

## Story 27: Get the count of total reviews for a submission

### Stakeholder Request
*"We want organizers to immediately see how many reviews a talk has received. Please add a property returning the total scorecard count."*

### Junior Walkthrough & Explanation
We can check count sizes on reverse foreign key collections:

### Detailed Implementation Steps
1.  **Locate Submission Model:** Open `src/pretalx/submission/models/submission.py`.
2.  **Add Count Property:**
    ```python
    @property
    def review_count(self) -> int:
        return self.reviews.count()
    ```

---

## Story 28: Add a property `is_confirmed` to the `Submission` model

### Stakeholder Request
*"We often check if speakers have clicked 'Confirm Attendance' on accepted talks. Please add a simple helper property to check this state."*

### Junior Walkthrough & Explanation
Let's check if the state matches `CONFIRMED`:

### Detailed Implementation Steps
1.  **Locate Submission Model:** Open `src/pretalx/submission/models/submission.py`.
2.  **Add Property:**
    ```python
    @property
    def is_confirmed(self) -> bool:
        return self.state == SubmissionStates.CONFIRMED
    ```

---

## Story 29: Display track names next to talk slots in schedules

### Stakeholder Request
*"When looking at published schedules, all talks look identical. Let's print the track name next to the session titles."*

### Junior Walkthrough & Explanation
We locate the public agenda scheduling template and render track properties from the slot's talk instance.

### Detailed Implementation Steps
1.  **Locate Agenda Template:** Open `src/pretalx/agenda/templates/agenda/schedule.html`.
2.  **Inject track context markup:**
    ```html
    {% if slot.submission.track %}
        <span class="badge badge-track">{{ slot.submission.track.name }}</span>
    {% endif %}
    ```

---

## Story 30: Order rooms in visual schedules by capacity

### Stakeholder Request
*"In the schedule dashboard view, rooms are listed randomly. Please sort room listings by capacity (largest room first) to make allocation easier."*

### Junior Walkthrough & Explanation
We change the default sorting parameter in the room model.

### Detailed Implementation Steps
1.  **Locate Room Model:** Open `src/pretalx/schedule/models/room.py`.
2.  **Add Ordering settings:**
    ```python
    class Meta:
        ordering = ("-capacity",)  # '-' specifies descending order
    ```
