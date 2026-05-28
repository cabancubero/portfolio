# Backend - Portfolio Breakdown

## Skills Demonstrated

| Skill | Where It Appears |
|-------|-------------------|
| **Python** | Core language across all backend modules |
| **Django / Django REST Framework** | Full API built on DRF with ViewSets, serializers, custom permissions, and router-based URL configuration |
| **PostgreSQL** | Primary database with indexed fields, unique constraints, and relational modeling |
| **RESTful API Design** | Resource-oriented endpoints with consistent CRUD patterns, custom actions, filtering, search, and ordering |
| **Database Modeling & ORM** | Foreign keys, M2M relationships, through models, `unique_together` constraints, model properties, and `save()` overrides |
| **Authentication & Authorization** | Token + session auth with custom object-level permissions (`IsSelfOrAdmin`, `IsOwnerOrReadOnly`, `IsAuthorOrReadOnly`) |
| **Docker** | Multi-service containerization with separate dev and production configurations (Gunicorn, Nginx, PostgreSQL) |
| **Data Validation** | Custom serializer validation with structured error messages and range checking |
| **Transaction Management** | `transaction.atomic()` for multi-step operations that must succeed or fail together |

---

## Architectural Decisions

### 1. Multi-Step Onboarding as a State Machine
User onboarding is modeled as a series of boolean completion flags (`issues_completed`, `issues_ranked`, `actions_completed`, `values_completed`, `profile_completed`) on a single `UserOnboarding` model. The final `complete` endpoint checks all flags before marking `is_complete = True`. This design allows users to resume onboarding from any step, supports idempotent re-submission (each step clears previous selections before saving new ones), and gives the frontend a single status object to determine which screen to show.

### 2. Separation of Read and Write Serializers
ViewSets like `LearningResourceViewSet` and `LearningJourneyViewSet` use `get_serializer_class()` to return different serializers depending on the action. Read serializers include nested related objects for display; write serializers accept only IDs and validated input. This prevents over-exposure of data on create/update and keeps each serializer focused on one job.

### 3. Domain-Specific Apps with a Shared Category System
Rather than one monolithic app, the backend is split into focused Django apps (`events`, `communities`, `messaging`, `learning`, `onboarding`, `matching`, `organizations`). All share a centralized `categories` app for cross-domain categorization - events, community posts, learning resources, and organizations all reference the same `Category` model. This avoids duplication while keeping each domain's logic isolated.

---

## Code Snippets (Core Competencies)

### 1. Onboarding State Machine - Atomic Ranking with Step Tracking
**Skills:** Django ORM, transaction management, state machine pattern, DRF custom actions

Each onboarding step validates preconditions, wraps mutations in an atomic transaction, and updates a completion flag. The `complete` endpoint enforces that all steps must pass before finalizing.

```python
@action(detail=False, methods=['post'])
def rank(self, request):
    serializer = IssueRankSerializer(data=request.data)
    if serializer.is_valid():
        user_onboarding = get_object_or_404(UserOnboarding, user=request.user)

        if not user_onboarding.selected_issues.exists():
            return Response(
                {"detail": "You must select issues before ranking them"},
                status=status.HTTP_400_BAD_REQUEST
            )

        with transaction.atomic():
            for ranking in serializer.validated_data['rankings']:
                issue_id = ranking['issue_id']
                rank = ranking['rank']
                user_issue = get_object_or_404(
                    UserIssue,
                    user_onboarding=user_onboarding,
                    issue_id=issue_id
                )
                user_issue.rank = rank
                user_issue.save()

            user_onboarding.issues_ranked = True
            user_onboarding.save()

        return Response(
            {"detail": "Issue rankings updated successfully"},
            status=status.HTTP_200_OK
        )
    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

@action(detail=False, methods=['put'])
def complete(self, request):
    user_onboarding = self.get_object()

    if (user_onboarding.issues_completed and
        user_onboarding.issues_ranked and
        user_onboarding.actions_completed and
        user_onboarding.values_completed and
        user_onboarding.profile_completed):

        user_onboarding.is_complete = True
        user_onboarding.completed_at = timezone.now()
        user_onboarding.save()
        return Response({"detail": "Onboarding completed successfully"},
                        status=status.HTTP_200_OK)
    else:
        return Response({"detail": "All onboarding steps must be completed first"},
                        status=status.HTTP_400_BAD_REQUEST)
```

### 2. Community Voting System - Idempotent Toggle with M2M
**Skills:** DRF custom actions, M2M relationship management, model properties

The vote endpoint uses a clear-then-set pattern: always remove existing votes first, then apply the new one. This makes the operation idempotent - calling it multiple times with the same input produces the same result. The `vote_score` is a computed model property, not a stored field, so it's always consistent.

```python
# Model
class Community(models.Model):
    upvotes = models.ManyToManyField(settings.AUTH_USER_MODEL,
                                     related_name='upvoted_posts', blank=True)
    downvotes = models.ManyToManyField(settings.AUTH_USER_MODEL,
                                       related_name='downvoted_posts', blank=True)

    @property
    def vote_score(self):
        return self.upvotes.count() - self.downvotes.count()

# View
@action(detail=True, methods=['post'])
def vote(self, request, pk=None):
    post = self.get_object()
    user = request.user
    vote_type = request.data.get('vote_type')

    if vote_type not in ['up', 'down', 'none']:
        return Response({"detail": "Vote type must be 'up', 'down', or 'none'."},
                        status=status.HTTP_400_BAD_REQUEST)

    post.upvotes.remove(user)
    post.downvotes.remove(user)

    if vote_type == 'up':
        post.upvotes.add(user)
    elif vote_type == 'down':
        post.downvotes.add(user)

    return Response({"vote_score": post.vote_score})
```

### 3. Matching System Data Model - Algorithm-to-Database Bridge
**Skills:** Database design, Django ORM constraints, indexed queries, `save()` override

The matching models mirror the algorithm's data structures (issues, actions, values, weights) as database tables. The `MatchResult` model caches computed scores with composite indexes for fast lookup. Each model overrides `save()` to keep a denormalized `organization_identifier` in sync, enabling indexed lookups without joins.

```python
class OrganizationIssueRanking(models.Model):
    organization = models.ForeignKey(Organization,
                                     on_delete=models.CASCADE, related_name='issue_rankings')
    organization_identifier = models.CharField(max_length=20, db_index=True)
    issue = models.ForeignKey(Issue, on_delete=models.CASCADE)
    rank = models.PositiveIntegerField()

    class Meta:
        unique_together = [['organization', 'issue']]
        ordering = ['rank']
        indexes = [models.Index(fields=['organization_identifier'])]

    def save(self, *args, **kwargs):
        if self.organization_id:
            self.organization_identifier = self.organization.identifier
        super().save(*args, **kwargs)

class MatchResult(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL,
                             on_delete=models.CASCADE, related_name='match_results')
    organization = models.ForeignKey(Organization,
                                     on_delete=models.CASCADE, related_name='match_results')
    organization_identifier = models.CharField(max_length=20, db_index=True)
    issue_score = models.FloatField()
    value_score = models.FloatField()
    action_score = models.FloatField()
    total_score = models.FloatField()
    timestamp = models.DateTimeField(auto_now=True)

    class Meta:
        unique_together = [['user', 'organization']]
        get_latest_by = 'timestamp'
        indexes = [
            models.Index(fields=['user', 'total_score']),
            models.Index(fields=['organization_identifier'])
        ]
```

### 4. Conversation Serializer with Computed Read State
**Skills:** DRF serializer method fields, request context, queryset filtering

The conversation serializer computes `unread_count` per-user at serialization time by excluding messages the user sent or already read. The `last_message` is a model property using reverse relation ordering. This keeps the API response rich without requiring additional endpoints.

```python
class ConversationSerializer(serializers.ModelSerializer):
    participants = UserMinimalSerializer(many=True, read_only=True)
    participant_ids = serializers.PrimaryKeyRelatedField(
        many=True, write_only=True,
        queryset=User.objects.all(), source='participants'
    )
    last_message = LastMessageSerializer(read_only=True)
    unread_count = serializers.SerializerMethodField()

    class Meta:
        model = Conversation
        fields = ['id', 'name', 'participants', 'participant_ids',
                  'created_at', 'updated_at', 'is_group', 'last_message',
                  'unread_count']
        read_only_fields = ['created_at', 'updated_at']

    def get_unread_count(self, obj):
        user = self.context['request'].user
        return obj.messages.exclude(read_by=user).exclude(sender=user).count()
```

### 5. Trending Events - Database-Level Scoring with Annotations
**Skills:** Django ORM annotations, `ExpressionWrapper`, query parameter handling, `F` expressions

The trending endpoint scores events by view count and recency entirely in the database using Django ORM annotations. It filters by a configurable time window, enforces a minimum view threshold, and annotates with age for ranking - no Python-level sorting needed.

```python
@action(detail=False, methods=['get'])
def trending(self, request):
    time_window = request.query_params.get('days', 7)
    try:
        time_window = int(time_window)
    except ValueError:
        time_window = 7

    cutoff_date = timezone.now() - timedelta(days=time_window)
    min_views = 5

    queryset = Event.objects.filter(
        last_viewed_at__gte=cutoff_date,
        view_count__gte=min_views
    )

    now = timezone.now()
    queryset = queryset.annotate(
        days_since_creation=ExpressionWrapper(
            (now - F('created_at')),
            output_field=fields.DurationField()
        )
    )

    queryset = queryset.order_by('-view_count', 'days_since_creation')

    limit = request.query_params.get('limit', 10)
    try:
        limit = int(limit)
    except ValueError:
        limit = 10

    queryset = queryset[:limit]
    serializer = self.get_serializer(queryset, many=True)
    return Response(serializer.data)
```

---

## Clever Implementations

### 1. Conversation Deduplication in the Serializer's `create()`
When a user starts a 1-on-1 conversation, the serializer checks whether a non-group conversation already exists between those two participants before creating a new one. If it finds one, it returns the existing conversation instead. This prevents the common messaging bug where two users end up with multiple parallel conversation threads, and it handles it at the data layer rather than requiring frontend coordination.

```python
def create(self, validated_data):
    user = self.context['request'].user
    participants = list(validated_data.get('participants', []))

    if user not in participants:
        participants.append(user)
        validated_data['participants'] = participants

    is_group = len(participants) > 2 or 'name' in validated_data
    validated_data['is_group'] = is_group

    if not is_group and len(participants) == 2:
        other_user = next(p for p in participants if p != user)
        existing_conversations = Conversation.objects.filter(is_group=False)

        for existing_conv in existing_conversations:
            if existing_conv.participants.count() == 2:
                existing_participants = list(existing_conv.participants.all())
                if user in existing_participants and other_user in existing_participants:
                    return existing_conv

    conversation = super().create(validated_data)
    return conversation
```

The logic also auto-classifies conversations: if there are more than 2 participants or a name is provided, it's flagged as a group conversation. Otherwise it's treated as a direct message.

### 2. Human-Readable "Last Seen" as a Computed Property
The `UserStatus` model stores raw `is_online` and `last_active` timestamps, but exposes a `status_text` property that converts these into human-readable strings like "last seen 3 hours ago" or "last seen yesterday". This keeps the display logic server-side so every client gets the same text, and it degrades through progressively coarser time buckets (seconds, minutes, hours, days) rather than showing raw timestamps.

```python
class UserStatus(models.Model):
    user = models.OneToOneField(settings.AUTH_USER_MODEL,
                                on_delete=models.CASCADE, related_name='status')
    is_online = models.BooleanField(default=False)
    last_active = models.DateTimeField(default=timezone.now)

    @property
    def status_text(self):
        if self.is_online:
            return "online"

        time_diff = timezone.now() - self.last_active
        if time_diff.days > 0:
            if time_diff.days == 1:
                return "last seen yesterday"
            return f"last seen {time_diff.days} days ago"
        elif time_diff.seconds > 3600:
            hours = time_diff.seconds // 3600
            return f"last seen {hours} hour{'s' if hours > 1 else ''} ago"
        elif time_diff.seconds > 60:
            minutes = time_diff.seconds // 60
            return f"last seen {minutes} minute{'s' if minutes > 1 else ''} ago"
        else:
            return "last seen just now"
```
