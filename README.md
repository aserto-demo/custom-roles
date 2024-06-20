# Custom roles

This sample contains the [Topaz](https://www.topaz.sh) manifest and data for modeling an RBAC system with a set of built-in roles, and the ability to add custom roles.

## Installing the sample

1. Make sure you have Topaz installed:

```bash
brew tap aserto-dev/topaz && brew install topaz
```

2. Create a new configuration called "custom-roles"

```bash
topaz templates install simple-rbac --config-name="custom-roles"
```

3. Overwrite the manifest with the one in this repo:

```bash
topaz directory set manifest ./model/manifest.yaml
```

4. Import the sample objects and relations

```bash
topaz directory import -d ./data
```

5. View the data in the Topaz console

```bash
topaz console
```

## Model and data

### System roles
In this simple example, we have three simple "system" roles for each tenant - `admin`, `member`, and `viewer`.

### Permissions
These roles grant the following permissions:
* `can_read` - granted through `admin`, `member`, and `viewer`
* `can_write` - granted through `admin` and `member`
* `can_admin` - granted through `admin`
* `can_add_billing_info` - granted through `admin`

### Custom roles
The requirement to allow tenants to have custom roles is satisfied by treating roles and permissions as object types.

### Model
The model defines the following object types: `user`, `group`, `role`, `permission`.

The relevant parts of the model are shown below:

```yaml
types:
  user: {}

  group:
    relations:
      member: user | group#member

  role:
    relations:
      member: group#member
    permissions:
      has_permission: member

  permission:
    relations:
      role: role
    permissions:
      has_permission: role | role->has_permission
```

### Data
Permission object instances are singletons, and correspond to the permissions in the system:
* `{ "object_type": "permission", "object_id": "can-read" }`
* `{ "object_type": "permission", "object_id": "can-write" }`
* `{ "object_type": "permission", "object_id": "can-delete" }`
* `{ "object_type": "permission", "object_id": "can-add-billing-info" }`

For a multi-tenant system, when a tenant is created, the following object instances and relationships should be created:

System Roles for each tenant:
* `{ "object_type": "role", "object_id": "<tenant-name>-admin" }`
* `{ "object_type": "role", "object_id": "<tenant-name>-member" }`
* `{ "object_type": "role", "object_id": "<tenant-name>-viewer }"`

System Groups for each tenant:
* `{ "object_type": "group", "object_id": "<tenant-name>-admin" }`
* `{ "object_type": "group", "object_id": "<tenant-name>-member" }`
* `{ "object_type": "group", "object_id": "<tenant-name>-viewer }"`

Relationships:
* Relationships between the permissions and each system role as detailed in the "Permissions" section.
* Relationships between the system roles and system groups.
* Relationships between users and the system groups.

The data that is loaded creates two such tenants: Acmecorp and Fabrikam.

A couple of examples:

* Acmecorp has an `acmecorp-admin` role, which includes all permissions. As its target it has the `acmecorp-admin` group, which includes the user `rick@the-citadel.com`.
* Fabrikam has a `fabrikam-admin` role, which includes all permissions. As its target it has the `fabrikam-admin` group, which includes the user `beth@the-citadel.com`.

### Custom roles
The Acmecorp tenant has an additional `role` object called `acmecorp-billing-admin`. This Role ONLY has the `can_add_billing_info` permission. The Role also has a corresponding group called `acmecorp-billing-admin`. The user `morty@the-citadel.com` is a member of this group.

## Testing

### Does Rick have the `can_delete` and `can_add_billing_info` permissions?

Since Rick is a member of the `acmecorp-admin` group, he should have both the `can_delete` and `can_add_billing_info` permissions. 

```bash
topaz ds check '
{
  "object_type": "permission",
  "object_id": "can_delete",
  "relation": "has_permission",
  "subject_type": "user",
  "subject_id": "rick@the-citadel.com"
}'
{
  "check":  true,
  "trace":  []
}
```

```bash
topaz ds check '
{
  "object_type": "permission",
  "object_id": "can_add_billing_info",
  "relation": "has_permission",
  "subject_type": "user",
  "subject_id": "rick@the-citadel.com"
}'
{
  "check":  true,
  "trace":  []
}
```

### Does Morty have the `can_read` and `can_add_billing_info` permissions?

Since Morty is a member of the `acmecorp-billing-admin` group, he only has the `can_add_billing_info` permission. 

```bash
topaz ds check '
{
  "object_type": "permission",
  "object_id": "can_read",
  "relation": "has_permission",
  "subject_type": "user",
  "subject_id": "morty@the-citadel.com"
}'
{
  "check":  false,
  "trace":  []
}
```

```bash
topaz ds check '
{
  "object_type": "permission",
  "object_id": "can_add_billing_info",
  "relation": "has_permission",
  "subject_type": "user",
  "subject_id": "morty@the-citadel.com"
}'
{
  "check":  true,
  "trace":  []
}
```

### Adding a new custom role

Let's imagine the Fabrikam tenant wants a custom role that gives a user the `can_add_billing_info` AND `can_view` permissions on the tenant.

To add this custom role to the Fabrikam tenant, you'll need to:

1. Create the role object:

```bash
topaz ds set object '
{ "object":
  {
    "type": "role",
    "id": "fabrikam-custom-role",
    "display_name": "Fabrikam Custom Role"
  }
}'
```

2. Create the group object:
```bash
topaz ds set object '
{ "object":
  {
    "type": "group",
    "id": "fabrikam-custom-role",
    "display_name": "Fabrikam Custom Role Group"
  }
}'
```

3. Relate the permissions to the custom role:
```bash
topaz ds set relation '
{
  "relation":  {
    "object_type": "permission",
    "object_id":  "can_read",
    "relation":  "role",
    "subject_type":  "role",
    "subject_id":  "fabrikam-custom-role"
  }
}'
```

```bash
topaz ds set relation '
{
  "relation":  {
    "object_type": "permission",
    "object_id":  "can_add_billing_info",
    "relation":  "role",
    "subject_type":  "role",
    "subject_id":  "fabrikam-custom-role"
  }
}'
```

4. Relate the custom role to the custom group:
```bash
topaz ds set relation '
{
  "relation":  {
    "object_type": "role",
    "object_id":  "fabrikam-custom-role",
    "relation":  "member",
    "subject_type":  "group",
    "subject_id":  "fabrikam-custom-role",
    "subject_relation": "member"
  }
}'
```

5. Relate a user (Morty) to the custom group:
```bash
topaz ds set relation '
{
  "relation":  {
    "object_type": "group",
    "object_id":  "fabrikam-custom-role",
    "relation":  "member",
    "subject_type":  "user",
    "subject_id":  "morty@the-citadel.com"
  }
}'
```

Now Morty should have the `can_read` permission as well:

```bash
topaz ds check '
{
  "object_type": "permission",
  "object_id": "can_read",
  "relation": "has_permission",
  "subject_type": "user",
  "subject_id": "morty@the-citadel.com"
}'
{
  "check":  true,
  "trace":  []
}
```
