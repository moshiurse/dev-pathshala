# 🛡️ Fine-grained Authorization (RBAC, ABAC, PBAC, ReBAC)

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [আর্কিটেকচার ডায়াগ্রাম](#আর্কিটেকচার-ডায়াগ্রাম)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

### Fine-grained Authorization কী?

**Fine-grained Authorization** হলো এমন একটি access control system যেখানে শুধু "এই user কি login করেছে?" না জিজ্ঞেস করে বরং জিজ্ঞেস করে — "এই user, এই specific resource-এ, এই specific action, এই specific context-এ করতে পারবে কিনা?"

### চার ধরনের Authorization Model:

| Model | Full Name | সিদ্ধান্তের ভিত্তি | উদাহরণ |
|-------|-----------|---------------------|---------|
| **RBAC** | Role-Based Access Control | User-এর Role | Admin পারে, Viewer পারে না |
| **ABAC** | Attribute-Based Access Control | User/Resource/Environment attributes | ঢাকা অফিস থেকে, অফিস আওয়ারে |
| **PBAC** | Policy-Based Access Control | Centralized policy rules (OPA) | Policy engine decide করবে |
| **ReBAC** | Relationship-Based Access Control | Objects-এর মধ্যে relationship | Owner, Editor, Viewer |

### RBAC (Role-Based Access Control):

```
User ──belongs_to──▶ Role ──has──▶ Permission

উদাহরণ:
User: "করিম"
├── Role: "teacher"
│   ├── Permission: "view_grades"
│   ├── Permission: "edit_grades"
│   └── Permission: "view_students"
└── Role: "class_advisor"
    ├── Permission: "view_attendance"
    └── Permission: "approve_leave"

Role Hierarchy:
    super_admin
        ├── admin
        │   ├── teacher
        │   │   └── viewer
        │   └── accountant
        └── principal
```

### ABAC (Attribute-Based Access Control):

```
Decision = f(Subject attributes, Resource attributes, Action, Environment)

Subject Attributes:    Resource Attributes:    Environment:
├── role: teacher      ├── type: grade         ├── time: 10:00 AM
├── department: science├── class: 10-A         ├── ip: office_range
├── tenure: 5 years    ├── owner: teacher_123  ├── device: registered
└── clearance: high    └── sensitivity: medium └── location: Dhaka

Rule: "teacher can edit grades IF grade.class IN teacher.assigned_classes 
       AND environment.time IN office_hours"
```

### PBAC (Policy-Based Access Control — OPA/Rego):

```
Policy Decision Point (PDP) — OPA Server:
  Input: {user, action, resource, context}
  Policy: Rego rules
  Output: {allow: true/false, reasons: [...]}

Policy Enforcement Point (PEP) — Application/API Gateway:
  Request → Ask PDP → Allow/Deny → Respond
```

### ReBAC (Relationship-Based Access Control — Zanzibar):

```
Object#relation@subject

document:report-2024#owner@user:karim
document:report-2024#editor@team:science-dept#member
document:report-2024#viewer@school:abc-school#student

Check: "Can user:rahim view document:report-2024?"
Answer: Is rahim a member of school:abc-school? If yes → viewer → can view ✅
```

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🏫 School Management SaaS — Multi-tenant Authorization

```
System: "শিক্ষাবন্ধু" — Bangladesh School Management Platform
Tenants: ৫০০+ স্কুল ব্যবহার করে

Roles per School (RBAC):
├── Principal (অধ্যক্ষ)
│   ├── সব কিছু দেখতে ও edit করতে পারে (নিজ স্কুলে)
│   ├── Teacher appointment approve করতে পারে
│   └── Financial reports দেখতে পারে
│
├── Teacher (শিক্ষক)
│   ├── নিজের class-এর students দেখতে পারে
│   ├── Grade/attendance দিতে পারে (নিজের subject-এ)
│   └── অন্য class-এর data দেখতে পারে না
│
├── Student (ছাত্র/ছাত্রী)
│   ├── নিজের grades দেখতে পারে
│   ├── Assignment submit করতে পারে
│   └── অন্যের grade দেখতে পারে না
│
├── Parent (অভিভাবক)
│   ├── নিজের সন্তানের data দেখতে পারে
│   ├── Teacher-এর সাথে message করতে পারে
│   └── অন্য student-এর data দেখতে পারে না
│
└── Accountant (হিসাবরক্ষক)
    ├── Fee collection data দেখতে/edit করতে পারে
    ├── Financial reports generate করতে পারে
    └── Academic data-তে access নেই

ABAC Layer (additional):
- Teacher শুধু office hours-এ grade edit করতে পারে
- Principal শুধু approved device থেকে financial data দেখতে পারে
- Student exam period-এ attendance self-report করতে পারে না
```

### 🚗 Pathao-র ReBAC Model:

```
Entities ও Relationships:

ride:RIDE-001#passenger@user:rahim
ride:RIDE-001#driver@user:karim
ride:RIDE-001#belongs_to@city:dhaka

user:karim#member@team:drivers-dhaka
team:drivers-dhaka#operates_in@city:dhaka
city:dhaka#part_of@country:bangladesh

Authorization Checks:
Q: "Can user:rahim cancel ride:RIDE-001?"
A: rahim is passenger of RIDE-001 → passengers can cancel → ✅

Q: "Can user:rahim see driver:karim's phone?"
A: rahim is passenger of RIDE-001, karim is driver of RIDE-001 
   → passenger can see assigned driver's phone → ✅

Q: "Can user:rahim see ALL rides?"
A: rahim has no admin/support relationship → ❌
```

### 📰 Prothom Alo Content Management:

```
PBAC with OPA:

Policy: "article_publish"
Rules (Rego):
- Editor can publish articles in their assigned section
- Senior Editor can publish in any section
- Breaking news can be published by any Editor regardless of section
- Article must have at least 2 reviews before publish
- After 10 PM, only Night Desk Editor can publish

Input to OPA:
{
  "user": {"role": "editor", "section": "sports", "shift": "day"},
  "resource": {"type": "article", "section": "sports", "reviews": 3},
  "action": "publish",
  "environment": {"time": "14:30", "day": "weekday"}
}

Output: {"allow": true, "reason": "editor can publish reviewed article in own section during shift"}
```

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### Overall Authorization Architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Request                            │
│  User: teacher_karim | Action: edit_grade | Resource: grade_123  │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                   API Gateway / PEP                                │
│              (Policy Enforcement Point)                            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  1. Extract user identity (JWT)                             │  │
│  │  2. Extract action & resource from request                  │  │
│  │  3. Ask PDP: "Can this user do this action on this resource?"│ │
│  │  4. Allow or Deny based on PDP response                     │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                    PDP (Policy Decision Point)                     │
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │    RBAC     │  │    ABAC     │  │   ReBAC (SpiceDB)       │ │
│  │   Engine    │  │   Engine    │  │   Relationship Check    │ │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘ │
│         │                │                      │                │
│         └────────────────┼──────────────────────┘                │
│                          │                                        │
│                          ▼                                        │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              Policy Combiner / OPA                          │  │
│  │  RBAC: allow? ✅  ABAC: allow? ✅  ReBAC: allow? ✅       │  │
│  │  Final Decision: ALLOW (all engines agree)                  │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### RBAC Role Hierarchy:

```
                    ┌──────────────────┐
                    │   Super Admin    │
                    │ (সর্বোচ্চ ক্ষমতা) │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼────────┐ ┌──▼───────┐ ┌───▼─────────┐
     │   Principal     │ │  Admin   │ │ Super       │
     │ (অধ্যক্ষ)       │ │ (IT)     │ │ Accountant  │
     └────────┬────────┘ └──────────┘ └─────────────┘
              │
    ┌─────────┼─────────────┐
    │         │             │
┌───▼────┐ ┌─▼──────────┐ ┌▼───────────┐
│Teacher │ │Accountant  │ │Librarian   │
│(শিক্ষক)│ │(হিসাবরক্ষক)│ │(গ্রন্থাগারিক)│
└───┬────┘ └────────────┘ └────────────┘
    │
┌───▼──────┐
│ Viewer   │
│(শুধু দেখা)│
└──────────┘

Inheritance: Teacher inherits all Viewer permissions
             Principal inherits all Teacher + Accountant permissions
```

### ReBAC Relationship Graph (Zanzibar Model):

```
┌────────────────────────────────────────────────────────────┐
│                    Relationship Graph                        │
│                                                             │
│   school:abc-school                                         │
│        │                                                    │
│        ├──member──▶ user:principal_rashid                   │
│        ├──member──▶ user:teacher_karim                      │
│        ├──member──▶ user:student_rahim                      │
│        └──member──▶ user:parent_salma                       │
│                                                             │
│   class:10-A                                                │
│        │                                                    │
│        ├──belongs_to──▶ school:abc-school                   │
│        ├──teacher──▶ user:teacher_karim                     │
│        ├──student──▶ user:student_rahim                     │
│        └──student──▶ user:student_nadia                     │
│                                                             │
│   grade:rahim-math-2024                                     │
│        │                                                    │
│        ├──owner──▶ user:teacher_karim                       │
│        ├──subject──▶ user:student_rahim                     │
│        ├──parent_of_subject──▶ user:parent_salma            │
│        └──belongs_to──▶ class:10-A                          │
│                                                             │
│   Permission computation:                                    │
│   "Can parent_salma view grade:rahim-math-2024?"            │
│   parent_salma ──parent_of_subject──▶ grade:rahim-math-2024│
│   parent_of_subject can "view" → ✅ ALLOW                   │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Authorization Decision Caching:

```
┌────────────┐     ┌──────────────┐     ┌─────────────┐
│  Service   │────▶│  Auth Cache  │────▶│     PDP     │
│  (PEP)     │     │   (Redis)    │     │  (OPA/     │
└────────────┘     └──────────────┘     │   SpiceDB) │
                                         └─────────────┘

Cache Strategy:
┌─────────────────────────────────────────────────────┐
│ Key: "user:karim|action:edit|resource:grade:123"    │
│ Value: {allow: true, ttl: 60s}                      │
│                                                      │
│ Invalidation triggers:                               │
│ ├── Role change → invalidate user's all cache       │
│ ├── Resource permission change → invalidate resource│
│ ├── Policy update → invalidate ALL cache            │
│ └── TTL expire → natural invalidation              │
└─────────────────────────────────────────────────────┘

Performance at Scale:
─────────────────────────────────────
Without cache: 50ms per auth check × 10K req/s = bottleneck
With cache:    <1ms (cache hit) for 95% requests
Cache miss:    50ms (PDP call) for 5% requests
Effective:     ~3.5ms average latency
```

### ABAC Policy Evaluation:

```
┌──────────────────────────────────────────────────────────────┐
│                     ABAC Policy Engine                         │
│                                                               │
│  Subject Attributes    Resource Attributes    Environment     │
│  ┌───────────────┐    ┌────────────────┐    ┌────────────┐  │
│  │ role: teacher │    │ type: grade    │    │ time: 10AM │  │
│  │ dept: science │    │ class: 10-A   │    │ ip: office │  │
│  │ school: abc   │    │ subject: math │    │ day: Mon   │  │
│  └───────┬───────┘    └───────┬────────┘    └──────┬─────┘  │
│          │                    │                     │         │
│          └────────────────────┼─────────────────────┘         │
│                               │                               │
│                               ▼                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                  Policy Rules                            │ │
│  │                                                          │ │
│  │  Rule 1: subject.role == "teacher"              ✅       │ │
│  │  Rule 2: resource.class IN subject.classes      ✅       │ │
│  │  Rule 3: environment.time IN "08:00-16:00"     ✅       │ │
│  │  Rule 4: environment.ip IN allowed_ranges       ✅       │ │
│  │                                                          │ │
│  │  All rules pass → ALLOW                                  │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Complete Multi-model Authorization System:

```php
<?php

namespace ShikkhaBondhu\Authorization;

/**
 * RBAC Engine — Role ভিত্তিক access control
 */
class RBACEngine
{
    private array $roleHierarchy = [];
    private array $rolePermissions = [];
    private array $userRoles = [];

    public function defineRole(string $role, array $permissions, ?string $parent = null): void
    {
        $this->rolePermissions[$role] = $permissions;
        if ($parent) {
            $this->roleHierarchy[$role] = $parent;
        }
    }

    public function assignRole(string $userId, string $role, string $tenantId): void
    {
        $this->userRoles["{$tenantId}:{$userId}"][] = $role;
    }

    /**
     * Permission check — role hierarchy সহ
     */
    public function hasPermission(string $userId, string $permission, string $tenantId): bool
    {
        $key = "{$tenantId}:{$userId}";
        $roles = $this->userRoles[$key] ?? [];

        foreach ($roles as $role) {
            if ($this->roleHasPermission($role, $permission)) {
                return true;
            }
        }

        return false;
    }

    /**
     * Role hierarchy traverse করে permission check
     */
    private function roleHasPermission(string $role, string $permission): bool
    {
        // Direct permission check
        $permissions = $this->rolePermissions[$role] ?? [];
        if (in_array($permission, $permissions) || in_array('*', $permissions)) {
            return true;
        }

        // Parent role-এ check (inheritance)
        $parent = $this->roleHierarchy[$role] ?? null;
        if ($parent) {
            return $this->roleHasPermission($parent, $permission);
        }

        return false;
    }

    /**
     * User-এর সব effective permissions (inherited সহ)
     */
    public function getEffectivePermissions(string $userId, string $tenantId): array
    {
        $key = "{$tenantId}:{$userId}";
        $roles = $this->userRoles[$key] ?? [];
        $allPermissions = [];

        foreach ($roles as $role) {
            $allPermissions = array_merge(
                $allPermissions,
                $this->getAllPermissionsForRole($role)
            );
        }

        return array_unique($allPermissions);
    }

    private function getAllPermissionsForRole(string $role): array
    {
        $permissions = $this->rolePermissions[$role] ?? [];
        $parent = $this->roleHierarchy[$role] ?? null;

        if ($parent) {
            $permissions = array_merge($permissions, $this->getAllPermissionsForRole($parent));
        }

        return $permissions;
    }
}

/**
 * ABAC Engine — Attribute ভিত্তিক access control
 */
class ABACEngine
{
    private array $policies = [];

    /**
     * Policy যোগ করা
     */
    public function addPolicy(string $name, callable $condition): void
    {
        $this->policies[$name] = $condition;
    }

    /**
     * সব applicable policies evaluate করো
     */
    public function evaluate(
        array $subjectAttributes,
        string $action,
        array $resourceAttributes,
        array $environmentAttributes
    ): AuthorizationDecision {
        $context = new ABACContext(
            subject: $subjectAttributes,
            action: $action,
            resource: $resourceAttributes,
            environment: $environmentAttributes
        );

        $results = [];
        foreach ($this->policies as $name => $condition) {
            $result = $condition($context);
            $results[$name] = $result;

            // কোনো policy DENY বললে — deny wins
            if ($result === false) {
                return new AuthorizationDecision(
                    allowed: false,
                    reason: "Denied by policy: {$name}",
                    evaluatedPolicies: $results
                );
            }
        }

        // সব policy allow বা not-applicable → allow
        $hasAllow = in_array(true, $results, true);

        return new AuthorizationDecision(
            allowed: $hasAllow,
            reason: $hasAllow ? 'Allowed by matching policies' : 'No applicable policy found',
            evaluatedPolicies: $results
        );
    }
}

class ABACContext
{
    public function __construct(
        public readonly array $subject,
        public readonly string $action,
        public readonly array $resource,
        public readonly array $environment
    ) {}
}

class AuthorizationDecision
{
    public function __construct(
        public readonly bool $allowed,
        public readonly string $reason,
        public readonly array $evaluatedPolicies = []
    ) {}
}

/**
 * ReBAC Engine — Relationship ভিত্তিক access control (Zanzibar-style)
 */
class ReBACEngine
{
    private array $relationships = [];
    private array $permissionRules = [];

    /**
     * Relationship তৈরি করো
     * Format: object#relation@subject
     */
    public function addRelationship(string $object, string $relation, string $subject): void
    {
        $this->relationships[] = [
            'object' => $object,
            'relation' => $relation,
            'subject' => $subject,
        ];
    }

    /**
     * Permission rule define করো
     * "owner" relation → "view", "edit", "delete" permissions
     */
    public function definePermission(string $objectType, string $permission, array $allowedRelations): void
    {
        $this->permissionRules["{$objectType}#{$permission}"] = $allowedRelations;
    }

    /**
     * Authorization check — relationship graph traverse
     */
    public function check(string $subject, string $permission, string $object): bool
    {
        $objectType = explode(':', $object)[0];
        $key = "{$objectType}#{$permission}";
        $allowedRelations = $this->permissionRules[$key] ?? [];

        foreach ($allowedRelations as $relation) {
            if ($this->hasRelationship($object, $relation, $subject)) {
                return true;
            }

            // Indirect relationship check (through groups/teams)
            if ($this->hasIndirectRelationship($object, $relation, $subject)) {
                return true;
            }
        }

        return false;
    }

    private function hasRelationship(string $object, string $relation, string $subject): bool
    {
        foreach ($this->relationships as $rel) {
            if ($rel['object'] === $object &&
                $rel['relation'] === $relation &&
                $rel['subject'] === $subject) {
                return true;
            }
        }
        return false;
    }

    /**
     * Indirect relationship — group membership traverse
     */
    private function hasIndirectRelationship(string $object, string $relation, string $subject): bool
    {
        // object-এ কোন groups/teams আছে এই relation-এ?
        $relatedGroups = array_filter($this->relationships, function ($rel) use ($object, $relation) {
            return $rel['object'] === $object &&
                   $rel['relation'] === $relation &&
                   str_contains($rel['subject'], '#member');
        });

        foreach ($relatedGroups as $groupRel) {
            // "team:science-dept#member" format parse
            $groupId = explode('#', $groupRel['subject'])[0];

            // subject কি এই group-এর member?
            if ($this->hasRelationship($groupId, 'member', $subject)) {
                return true;
            }
        }

        return false;
    }

    /**
     * User-এর সব accessible resources list করো
     */
    public function listAccessibleResources(string $subject, string $permission, string $objectType): array
    {
        $accessible = [];
        $key = "{$objectType}#{$permission}";
        $allowedRelations = $this->permissionRules[$key] ?? [];

        foreach ($this->relationships as $rel) {
            if (in_array($rel['relation'], $allowedRelations)) {
                $relObjectType = explode(':', $rel['object'])[0];
                if ($relObjectType === $objectType) {
                    if ($rel['subject'] === $subject || $this->isSubjectInGroup($subject, $rel['subject'])) {
                        $accessible[] = $rel['object'];
                    }
                }
            }
        }

        return array_unique($accessible);
    }

    private function isSubjectInGroup(string $subject, string $groupRef): bool
    {
        if (!str_contains($groupRef, '#member')) return false;
        $groupId = explode('#', $groupRef)[0];
        return $this->hasRelationship($groupId, 'member', $subject);
    }
}

/**
 * Combined Authorization Service — সব model একসাথে
 */
class AuthorizationService
{
    private RBACEngine $rbac;
    private ABACEngine $abac;
    private ReBACEngine $rebac;
    private AuthorizationCache $cache;
    private AuditLogger $audit;

    public function __construct(
        RBACEngine $rbac,
        ABACEngine $abac,
        ReBACEngine $rebac,
        AuthorizationCache $cache,
        AuditLogger $audit
    ) {
        $this->rbac = $rbac;
        $this->abac = $abac;
        $this->rebac = $rebac;
        $this->cache = $cache;
        $this->audit = $audit;
    }

    /**
     * Unified authorization check
     */
    public function authorize(AuthorizationRequest $request): AuthorizationDecision
    {
        // Cache check
        $cacheKey = $this->buildCacheKey($request);
        $cached = $this->cache->get($cacheKey);
        if ($cached !== null) {
            return $cached;
        }

        // Step 1: RBAC — basic role permission check
        $rbacResult = $this->rbac->hasPermission(
            $request->userId,
            $request->action,
            $request->tenantId
        );

        if (!$rbacResult) {
            $decision = new AuthorizationDecision(
                allowed: false,
                reason: 'RBAC: insufficient role permissions'
            );
            $this->finalize($request, $decision, $cacheKey);
            return $decision;
        }

        // Step 2: ReBAC — resource relationship check
        $rebacResult = $this->rebac->check(
            "user:{$request->userId}",
            $request->action,
            $request->resourceId
        );

        if (!$rebacResult) {
            $decision = new AuthorizationDecision(
                allowed: false,
                reason: 'ReBAC: no relationship with resource'
            );
            $this->finalize($request, $decision, $cacheKey);
            return $decision;
        }

        // Step 3: ABAC — contextual/environmental check
        $abacResult = $this->abac->evaluate(
            $request->subjectAttributes,
            $request->action,
            $request->resourceAttributes,
            $request->environmentAttributes
        );

        $this->finalize($request, $abacResult, $cacheKey);
        return $abacResult;
    }

    private function buildCacheKey(AuthorizationRequest $request): string
    {
        return md5(json_encode([
            $request->userId,
            $request->action,
            $request->resourceId,
            $request->tenantId,
        ]));
    }

    private function finalize(AuthorizationRequest $request, AuthorizationDecision $decision, string $cacheKey): void
    {
        // Cache store
        $this->cache->set($cacheKey, $decision, ttl: 60);

        // Audit log
        $this->audit->log($request, $decision);
    }
}

class AuthorizationRequest
{
    public function __construct(
        public readonly string $userId,
        public readonly string $action,
        public readonly string $resourceId,
        public readonly string $tenantId,
        public readonly array $subjectAttributes = [],
        public readonly array $resourceAttributes = [],
        public readonly array $environmentAttributes = []
    ) {}
}

class AuthorizationCache
{
    private array $store = [];

    public function get(string $key): ?AuthorizationDecision
    {
        $entry = $this->store[$key] ?? null;
        if (!$entry || $entry['expires_at'] < time()) {
            return null;
        }
        return $entry['decision'];
    }

    public function set(string $key, AuthorizationDecision $decision, int $ttl = 60): void
    {
        $this->store[$key] = [
            'decision' => $decision,
            'expires_at' => time() + $ttl,
        ];
    }

    public function invalidateUser(string $userId): void
    {
        // User-এর সব cache entry remove
        $this->store = array_filter($this->store, function ($key) use ($userId) {
            return !str_contains($key, $userId);
        }, ARRAY_FILTER_USE_KEY);
    }

    public function invalidateAll(): void
    {
        $this->store = [];
    }
}

class AuditLogger
{
    public function log(AuthorizationRequest $request, AuthorizationDecision $decision): void
    {
        $entry = [
            'timestamp' => date('c'),
            'user_id' => $request->userId,
            'tenant_id' => $request->tenantId,
            'action' => $request->action,
            'resource' => $request->resourceId,
            'decision' => $decision->allowed ? 'ALLOW' : 'DENY',
            'reason' => $decision->reason,
        ];
        error_log(json_encode($entry));
    }
}

// === শিক্ষাবন্ধু Platform Setup ===

// RBAC Setup
$rbac = new RBACEngine();

// Role hierarchy define
$rbac->defineRole('viewer', ['view_own_profile']);
$rbac->defineRole('student', ['view_grades', 'submit_assignment', 'view_schedule'], 'viewer');
$rbac->defineRole('parent', ['view_child_grades', 'message_teacher'], 'viewer');
$rbac->defineRole('teacher', ['edit_grades', 'view_students', 'take_attendance', 'view_grades'], 'viewer');
$rbac->defineRole('accountant', ['manage_fees', 'view_financial_reports'], 'viewer');
$rbac->defineRole('principal', ['approve_leave', 'view_all_data', 'manage_teachers'], 'teacher');
$rbac->defineRole('super_admin', ['*']);

// Assign roles (tenant-specific)
$rbac->assignRole('teacher_karim', 'teacher', 'school_abc');
$rbac->assignRole('student_rahim', 'student', 'school_abc');
$rbac->assignRole('parent_salma', 'parent', 'school_abc');
$rbac->assignRole('principal_rashid', 'principal', 'school_abc');

// ABAC Policies
$abac = new ABACEngine();

$abac->addPolicy('office_hours_only', function (ABACContext $ctx) {
    if ($ctx->action === 'edit_grades') {
        $hour = (int)date('H');
        return $hour >= 8 && $hour <= 16; // 8AM - 4PM only
    }
    return null; // not applicable
});

$abac->addPolicy('own_class_only', function (ABACContext $ctx) {
    if ($ctx->subject['role'] === 'teacher' && str_starts_with($ctx->action, 'edit_')) {
        $teacherClasses = $ctx->subject['assigned_classes'] ?? [];
        $resourceClass = $ctx->resource['class'] ?? '';
        return in_array($resourceClass, $teacherClasses);
    }
    return null;
});

// ReBAC Setup
$rebac = new ReBACEngine();

// Relationships
$rebac->addRelationship('class:10-A', 'teacher', 'user:teacher_karim');
$rebac->addRelationship('class:10-A', 'student', 'user:student_rahim');
$rebac->addRelationship('grade:rahim-math', 'owner', 'user:teacher_karim');
$rebac->addRelationship('grade:rahim-math', 'subject', 'user:student_rahim');
$rebac->addRelationship('grade:rahim-math', 'parent_of_subject', 'user:parent_salma');

// Permission rules
$rebac->definePermission('grade', 'view', ['owner', 'subject', 'parent_of_subject']);
$rebac->definePermission('grade', 'edit', ['owner']);
$rebac->definePermission('class', 'view_students', ['teacher']);

// Authorization check
$authService = new AuthorizationService(
    rbac: $rbac,
    abac: $abac,
    rebac: $rebac,
    cache: new AuthorizationCache(),
    audit: new AuditLogger()
);

$decision = $authService->authorize(new AuthorizationRequest(
    userId: 'teacher_karim',
    action: 'edit_grades',
    resourceId: 'grade:rahim-math',
    tenantId: 'school_abc',
    subjectAttributes: [
        'role' => 'teacher',
        'assigned_classes' => ['10-A', '10-B'],
    ],
    resourceAttributes: [
        'class' => '10-A',
        'subject' => 'math',
    ],
    environmentAttributes: [
        'time' => date('H:i'),
        'ip' => $_SERVER['REMOTE_ADDR'] ?? '127.0.0.1',
    ]
));

echo $decision->allowed ? "✅ অনুমতি আছে" : "❌ অনুমতি নেই: {$decision->reason}";
```

---

## 🟨 JavaScript কোড উদাহরণ

### Complete Authorization System (Node.js):

```javascript
/**
 * RBAC Engine — Role-Based Access Control
 */
class RBACEngine {
    constructor() {
        this.roles = new Map();          // role → {permissions, parent}
        this.userRoles = new Map();      // "tenant:userId" → [roles]
    }

    defineRole(name, permissions, parent = null) {
        this.roles.set(name, { permissions, parent });
    }

    assignRole(userId, role, tenantId) {
        const key = `${tenantId}:${userId}`;
        const roles = this.userRoles.get(key) || [];
        roles.push(role);
        this.userRoles.set(key, roles);
    }

    removeRole(userId, role, tenantId) {
        const key = `${tenantId}:${userId}`;
        const roles = this.userRoles.get(key) || [];
        this.userRoles.set(key, roles.filter(r => r !== role));
    }

    hasPermission(userId, permission, tenantId) {
        const key = `${tenantId}:${userId}`;
        const roles = this.userRoles.get(key) || [];

        for (const role of roles) {
            if (this._roleHasPermission(role, permission)) {
                return true;
            }
        }
        return false;
    }

    getEffectivePermissions(userId, tenantId) {
        const key = `${tenantId}:${userId}`;
        const roles = this.userRoles.get(key) || [];
        const allPerms = new Set();

        for (const role of roles) {
            for (const perm of this._getAllPermissions(role)) {
                allPerms.add(perm);
            }
        }

        return [...allPerms];
    }

    _roleHasPermission(role, permission) {
        const roleData = this.roles.get(role);
        if (!roleData) return false;

        if (roleData.permissions.includes(permission) || roleData.permissions.includes('*')) {
            return true;
        }

        // Check parent (inheritance)
        if (roleData.parent) {
            return this._roleHasPermission(roleData.parent, permission);
        }

        return false;
    }

    _getAllPermissions(role) {
        const roleData = this.roles.get(role);
        if (!roleData) return [];

        let perms = [...roleData.permissions];
        if (roleData.parent) {
            perms = [...perms, ...this._getAllPermissions(roleData.parent)];
        }
        return perms;
    }
}

/**
 * ABAC Engine — Attribute-Based Access Control
 */
class ABACEngine {
    constructor() {
        this.policies = [];
    }

    addPolicy(name, condition, effect = 'allow') {
        this.policies.push({ name, condition, effect });
    }

    evaluate(subject, action, resource, environment) {
        const context = { subject, action, resource, environment };
        const results = [];

        for (const policy of this.policies) {
            try {
                const result = policy.condition(context);
                if (result === null || result === undefined) continue; // not applicable

                results.push({
                    policy: policy.name,
                    result,
                    effect: policy.effect,
                });

                // Explicit deny wins
                if (policy.effect === 'deny' && result === true) {
                    return {
                        allowed: false,
                        reason: `Denied by policy: ${policy.name}`,
                        evaluatedPolicies: results,
                    };
                }
            } catch (err) {
                results.push({ policy: policy.name, error: err.message });
            }
        }

        const hasAllow = results.some(r => r.effect === 'allow' && r.result === true);
        return {
            allowed: hasAllow,
            reason: hasAllow ? 'Allowed by matching policies' : 'No matching allow policy',
            evaluatedPolicies: results,
        };
    }
}

/**
 * ReBAC Engine — Relationship-Based Access Control (Zanzibar-style)
 */
class ReBACEngine {
    constructor() {
        this.tuples = [];           // [{object, relation, subject}]
        this.permissionDefs = {};   // "objectType#permission" → [relations]
    }

    /**
     * Relationship tuple যোগ করো
     */
    addRelationship(object, relation, subject) {
        this.tuples.push({ object, relation, subject });
    }

    removeRelationship(object, relation, subject) {
        this.tuples = this.tuples.filter(t =>
            !(t.object === object && t.relation === relation && t.subject === subject)
        );
    }

    /**
     * Permission definition: কোন relation কোন permission দেয়
     */
    definePermission(objectType, permission, allowedRelations) {
        this.permissionDefs[`${objectType}#${permission}`] = allowedRelations;
    }

    /**
     * Authorization check — "Can subject do permission on object?"
     */
    check(subject, permission, object) {
        const objectType = object.split(':')[0];
        const key = `${objectType}#${permission}`;
        const allowedRelations = this.permissionDefs[key] || [];

        for (const relation of allowedRelations) {
            // Direct relationship check
            if (this._hasDirectRelation(object, relation, subject)) {
                return { allowed: true, via: `direct:${relation}` };
            }

            // Indirect via group membership
            const indirect = this._checkIndirect(object, relation, subject);
            if (indirect) {
                return { allowed: true, via: `indirect:${indirect}` };
            }
        }

        return { allowed: false, via: null };
    }

    /**
     * User-এর সব accessible resources list করো
     */
    listObjects(subject, permission, objectType) {
        const key = `${objectType}#${permission}`;
        const allowedRelations = this.permissionDefs[key] || [];
        const accessible = new Set();

        for (const tuple of this.tuples) {
            if (!tuple.object.startsWith(`${objectType}:`)) continue;
            if (!allowedRelations.includes(tuple.relation)) continue;

            if (tuple.subject === subject) {
                accessible.add(tuple.object);
            } else if (this._isSubjectInGroup(subject, tuple.subject)) {
                accessible.add(tuple.object);
            }
        }

        return [...accessible];
    }

    _hasDirectRelation(object, relation, subject) {
        return this.tuples.some(t =>
            t.object === object && t.relation === relation && t.subject === subject
        );
    }

    _checkIndirect(object, relation, subject) {
        // Find groups that have this relation to the object
        const groupTuples = this.tuples.filter(t =>
            t.object === object && t.relation === relation && t.subject.includes('#member')
        );

        for (const gt of groupTuples) {
            const groupId = gt.subject.split('#')[0];
            if (this._hasDirectRelation(groupId, 'member', subject)) {
                return `${groupId}→${relation}`;
            }
        }

        return null;
    }

    _isSubjectInGroup(subject, groupRef) {
        if (!groupRef.includes('#member')) return false;
        const groupId = groupRef.split('#')[0];
        return this._hasDirectRelation(groupId, 'member', subject);
    }
}

/**
 * OPA Client — Policy-Based Access Control
 */
class OPAClient {
    constructor(opaUrl = 'http://localhost:8181') {
        this.opaUrl = opaUrl;
    }

    async evaluate(policyPath, input) {
        const response = await fetch(`${this.opaUrl}/v1/data/${policyPath}`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ input }),
        });

        const data = await response.json();
        return {
            allowed: data.result?.allow || false,
            reasons: data.result?.reasons || [],
        };
    }
}

/**
 * Combined Authorization Service
 * সব model একত্রিত করে unified decision
 */
class AuthorizationService {
    constructor({ rbac, abac, rebac, cache, auditLog }) {
        this.rbac = rbac;
        this.abac = abac;
        this.rebac = rebac;
        this.cache = cache || new Map();
        this.auditLog = auditLog || [];
    }

    /**
     * Unified authorization check
     */
    async authorize(request) {
        const { userId, action, resourceId, tenantId, attributes } = request;

        // Cache check
        const cacheKey = `${tenantId}:${userId}:${action}:${resourceId}`;
        if (this.cache.has(cacheKey)) {
            const cached = this.cache.get(cacheKey);
            if (cached.expiresAt > Date.now()) {
                return { ...cached.decision, fromCache: true };
            }
            this.cache.delete(cacheKey);
        }

        // Step 1: RBAC check
        const rbacAllowed = this.rbac.hasPermission(userId, action, tenantId);
        if (!rbacAllowed) {
            return this._finalize(request, {
                allowed: false,
                reason: 'RBAC: role does not have this permission',
                layer: 'RBAC',
            }, cacheKey);
        }

        // Step 2: ReBAC check (resource relationship)
        const rebacResult = this.rebac.check(`user:${userId}`, action, resourceId);
        if (!rebacResult.allowed) {
            return this._finalize(request, {
                allowed: false,
                reason: 'ReBAC: no relationship with this resource',
                layer: 'ReBAC',
            }, cacheKey);
        }

        // Step 3: ABAC check (contextual)
        const abacResult = this.abac.evaluate(
            attributes?.subject || {},
            action,
            attributes?.resource || {},
            attributes?.environment || {}
        );

        if (!abacResult.allowed) {
            return this._finalize(request, {
                allowed: false,
                reason: `ABAC: ${abacResult.reason}`,
                layer: 'ABAC',
            }, cacheKey);
        }

        // All layers passed
        return this._finalize(request, {
            allowed: true,
            reason: 'All authorization layers passed',
            layers: { rbac: true, rebac: rebacResult.via, abac: true },
        }, cacheKey);
    }

    _finalize(request, decision, cacheKey) {
        // Cache
        this.cache.set(cacheKey, {
            decision,
            expiresAt: Date.now() + 60000, // 60 seconds TTL
        });

        // Audit
        this.auditLog.push({
            timestamp: new Date().toISOString(),
            ...request,
            decision: decision.allowed ? 'ALLOW' : 'DENY',
            reason: decision.reason,
        });

        return decision;
    }

    /**
     * Cache invalidation — role/permission change হলে
     */
    invalidateUser(userId) {
        for (const [key] of this.cache) {
            if (key.includes(`:${userId}:`)) {
                this.cache.delete(key);
            }
        }
    }

    invalidateAll() {
        this.cache.clear();
    }
}

// === "শিক্ষাবন্ধু" School Management SaaS Setup ===

// RBAC Setup
const rbac = new RBACEngine();

rbac.defineRole('viewer', ['view_own_profile']);
rbac.defineRole('student', ['view_grades', 'submit_assignment', 'view_schedule'], 'viewer');
rbac.defineRole('parent', ['view_child_grades', 'message_teacher', 'view_attendance'], 'viewer');
rbac.defineRole('teacher', ['view_grades', 'edit_grades', 'view_students', 'take_attendance'], 'viewer');
rbac.defineRole('accountant', ['manage_fees', 'view_financial_reports', 'generate_invoices'], 'viewer');
rbac.defineRole('principal', ['view_all', 'manage_teachers', 'approve_leave', 'edit_grades'], 'teacher');
rbac.defineRole('super_admin', ['*']);

// Assign roles per school (tenant)
rbac.assignRole('teacher_karim', 'teacher', 'school_abc');
rbac.assignRole('student_rahim', 'student', 'school_abc');
rbac.assignRole('parent_salma', 'parent', 'school_abc');
rbac.assignRole('principal_rashid', 'principal', 'school_abc');
rbac.assignRole('accountant_fatema', 'accountant', 'school_abc');

// ABAC Policies
const abac = new ABACEngine();

// শুধু office hours-এ grade edit করা যাবে
abac.addPolicy('office_hours_for_grade_edit', (ctx) => {
    if (ctx.action !== 'edit_grades') return null; // not applicable
    const hour = new Date().getHours();
    return hour >= 8 && hour <= 16;
}, 'allow');

// Teacher শুধু নিজের assigned class-এ edit করতে পারবে
abac.addPolicy('own_class_only', (ctx) => {
    if (ctx.subject.role !== 'teacher') return null;
    if (!ctx.action.startsWith('edit_')) return null;
    const assignedClasses = ctx.subject.assignedClasses || [];
    return assignedClasses.includes(ctx.resource.class);
}, 'allow');

// Exam period-এ student attendance self-report করতে পারবে না
abac.addPolicy('no_attendance_during_exam', (ctx) => {
    if (ctx.action !== 'self_attendance' || ctx.subject.role !== 'student') return null;
    return !ctx.environment.isExamPeriod; // true = allow, false = deny during exam
}, 'deny');

// ReBAC Setup
const rebac = new ReBACEngine();

// Relationships
rebac.addRelationship('class:10-A', 'teacher', 'user:teacher_karim');
rebac.addRelationship('class:10-A', 'student', 'user:student_rahim');
rebac.addRelationship('class:10-A', 'student', 'user:student_nadia');
rebac.addRelationship('class:10-B', 'teacher', 'user:teacher_karim');

rebac.addRelationship('grade:rahim-math-2024', 'owner', 'user:teacher_karim');
rebac.addRelationship('grade:rahim-math-2024', 'subject', 'user:student_rahim');
rebac.addRelationship('grade:rahim-math-2024', 'parent_of_subject', 'user:parent_salma');

rebac.addRelationship('grade:nadia-science-2024', 'owner', 'user:teacher_karim');
rebac.addRelationship('grade:nadia-science-2024', 'subject', 'user:student_nadia');

// Team/Department relationships
rebac.addRelationship('team:science-dept', 'member', 'user:teacher_karim');
rebac.addRelationship('team:science-dept', 'member', 'user:teacher_amina');

// Permission definitions
rebac.definePermission('grade', 'view', ['owner', 'subject', 'parent_of_subject']);
rebac.definePermission('grade', 'edit_grades', ['owner']);
rebac.definePermission('class', 'view_students', ['teacher']);
rebac.definePermission('class', 'take_attendance', ['teacher']);

// Combined Authorization Service
const authService = new AuthorizationService({ rbac, abac, rebac });

// === Test Cases ===

async function runAuthorizationTests() {
    // Test 1: Teacher edits own student's grade ✅
    const result1 = await authService.authorize({
        userId: 'teacher_karim',
        action: 'edit_grades',
        resourceId: 'grade:rahim-math-2024',
        tenantId: 'school_abc',
        attributes: {
            subject: { role: 'teacher', assignedClasses: ['10-A', '10-B'] },
            resource: { class: '10-A', subject: 'math' },
            environment: { time: '10:00', isExamPeriod: false },
        },
    });
    console.log('Test 1 (Teacher edit own grade):', result1.allowed ? '✅' : '❌', result1.reason);

    // Test 2: Parent views child's grade ✅
    const result2 = await authService.authorize({
        userId: 'parent_salma',
        action: 'view_child_grades',
        resourceId: 'grade:rahim-math-2024',
        tenantId: 'school_abc',
        attributes: {
            subject: { role: 'parent' },
            resource: { class: '10-A' },
            environment: {},
        },
    });
    console.log('Test 2 (Parent view child grade):', result2.allowed ? '✅' : '❌', result2.reason);

    // Test 3: Student views other's grade ❌
    const result3 = await authService.authorize({
        userId: 'student_rahim',
        action: 'view_grades',
        resourceId: 'grade:nadia-science-2024',
        tenantId: 'school_abc',
        attributes: {
            subject: { role: 'student' },
            resource: { class: '10-A' },
            environment: {},
        },
    });
    console.log('Test 3 (Student view others grade):', result3.allowed ? '✅' : '❌', result3.reason);

    // Test 4: Teacher edits OTHER class grade ❌
    const result4 = await authService.authorize({
        userId: 'teacher_karim',
        action: 'edit_grades',
        resourceId: 'grade:other-class-grade',
        tenantId: 'school_abc',
        attributes: {
            subject: { role: 'teacher', assignedClasses: ['10-A', '10-B'] },
            resource: { class: '9-C', subject: 'english' }, // Not assigned class
            environment: {},
        },
    });
    console.log('Test 4 (Teacher edit other class):', result4.allowed ? '✅' : '❌', result4.reason);
}

runAuthorizationTests();
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### 📊 সব Model-এর তুলনা:

```
┌─────────┬──────────────────┬──────────────────┬────────────────┐
│ Feature │     RBAC         │      ABAC        │     ReBAC      │
├─────────┼──────────────────┼──────────────────┼────────────────┤
│ জটিলতা  │ ★★☆☆☆ সহজ       │ ★★★★☆ মধ্যম     │ ★★★★★ কঠিন    │
│ নমনীয়তা │ ★★☆☆☆ কম        │ ★★★★☆ বেশি      │ ★★★★★ সবচেয়ে │
│ পারফর্ম.│ ★★★★★ দ্রুত     │ ★★★☆☆ মধ্যম     │ ★★★☆☆ মধ্যম   │
│ Audit   │ ★★★☆☆            │ ★★★★☆            │ ★★★★★          │
│ Scale   │ Role explosion   │ Policy complex   │ Graph grows    │
│ Tools   │ Any framework    │ OPA, Casbin      │ SpiceDB,       │
│         │                  │                  │ Authzed        │
│ Use Case│ Simple apps      │ Context-aware    │ Google Docs    │
│         │ Admin panels     │ Multi-tenant     │ style sharing  │
└─────────┴──────────────────┴──────────────────┴────────────────┘
```

### ✅ কখন কোনটি ব্যবহার করবেন:

| পরিস্থিতি | সেরা Model | কারণ |
|-----------|-----------|------|
| Simple admin panel | RBAC | মাত্র কয়েকটি role দরকার |
| E-commerce (Daraz) | RBAC + ABAC | Role + geo/time restrictions |
| Document sharing (Google Docs style) | ReBAC | Owner/Editor/Viewer relationships |
| Multi-tenant SaaS | RBAC + ReBAC | Tenant isolation + resource-level |
| Healthcare | ABAC | Complex attribute-based rules |
| Financial compliance | PBAC (OPA) | Auditable, centralized policies |
| School management | RBAC + ABAC + ReBAC | All three needed |

### ❌ কখন ব্যবহার করবেন না:

| পরিস্থিতি | কারণ |
|-----------|------|
| Simple blog (admin/reader) | RBAC-ই যথেষ্ট, fine-grained অপ্রয়োজনীয় |
| Static website | কোনো authorization দরকার নেই |
| Single-user application | Access control-ই দরকার নেই |
| Prototype/MVP | পরে add করা যায়, শুরুতে overhead |
| ReBAC with no relationships | Graph empty হলে ReBAC অর্থহীন |

### 🎯 Best Practices:

```
Architecture:
├── PDP (Policy Decision Point) আলাদা service হিসেবে রাখুন
├── PEP (Enforcement) API gateway/middleware-এ রাখুন
├── Authorization data application database-এ রাখবেন না
├── Cache authorization decisions (TTL 30-60s)
└── Audit EVERY decision (allow ও deny উভয়)

Performance:
├── Cache hit ratio 90%+ লক্ষ্য করুন
├── Warm cache on service startup
├── Batch authorization checks যেখানে possible
├── Pre-compute permissions for listing APIs
└── Async invalidation (eventual consistency OK)

Security:
├── Default DENY — কোনো policy match না করলে deny
├── Principle of least privilege
├── Regular permission audit (unused permissions remove)
├── Separation of duties (policy writer ≠ policy enforcer)
└── Test authorization with negative cases (deny scenarios)
```

### 🔄 Migration Path:

```
Phase 1: Start with RBAC (সহজ, দ্রুত implement)
         └── Admin, User, Viewer roles

Phase 2: Add ABAC rules (context-aware হতে)
         └── Time-based, location-based restrictions

Phase 3: Add ReBAC (resource-level granularity)
         └── Document ownership, team membership

Phase 4: Centralize with OPA/SpiceDB (scale-এ)
         └── Single policy engine, unified audit

Timeline per phase: ২-৪ সপ্তাহ
```

---

## 📚 সারসংক্ষেপ

```
Fine-grained Authorization = "কে, কী, কোথায়, কখন, কেন" — সব প্রশ্নের উত্তর

মনে রাখুন:
├── RBAC = Role দিয়ে permission (সহজ, কিন্তু rigid)
├── ABAC = Attribute দিয়ে decision (flexible, কিন্তু complex)
├── ReBAC = Relationship দিয়ে access (Google Docs style)
├── PBAC = Central policy engine (OPA — audit-friendly)
│
├── PDP = "সিদ্ধান্ত নেওয়ার জায়গা" (OPA/SpiceDB)
├── PEP = "সিদ্ধান্ত enforce করার জায়গা" (API Gateway)
├── Cache = Performance-র জন্য আবশ্যক
└── Audit = Compliance ও debugging-র জন্য আবশ্যক

Bangladesh Context:
├── School SaaS → RBAC + ABAC + ReBAC (multi-tenant)
├── bKash/Nagad → RBAC + ABAC (financial compliance)
├── Pathao → ReBAC (ride/driver/passenger relationships)
├── Prothom Alo → PBAC (content publishing policies)
└── Daraz → RBAC + ABAC (seller/buyer/admin + geo rules)
```
