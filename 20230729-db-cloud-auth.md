# Meta
[meta]: #meta
- Name: Cloud Authentication for database access
- Start Date: 2030-07-29
- Author(s): [KlausVii](https://github.com/KlausVii)
- Status: Draft <!-- Acceptable values: Draft, Approved, On Hold, Superseded -->
- RFC Pull Request: (leave blank)
- Relevant Issues: https://github.com/openfga/openfga/issues/870
- Supersedes: "N/A"

# Summary
[summary]: #summary

Many users run OpenFGA in a cloud environment, and these cloud providers often provide alternate authentication methods for their databases,
such as [AWS RDS IAM](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.html) and [GCP Cloud SQL Go Connector](https://cloud.google.com/sql/docs/postgres/samples/cloud-sql-postgres-databasesql-connect-connector).
This RFC proposes a new authentication method that allows users to authenticate against a cloud provider, starting with AWS RDS support.

# Motivation
[motivation]: #motivation

- Why should we do this?
  - Enabling cloud authentication methods will allow users to deploy OpenFGA faster and more securely.
  - Personally at our workplace all our other services authenticate with RDS IAM and not being able to do so with OpenFGA adds friction.
  - I imagine there are similar issues [on other cloud providers](https://cloud.google.com/sql/docs/mysql/connect-admin-ip#connect-ssl).
- What use cases does it support?
  - Users will be able to use their cloud provider's authentication method to authenticate against their database. 
- What is the expected outcome?
  - openFGA supports AWS IAM.
  - it is easy to extend to other cloud providers. 

# What it is
[what-it-is]: #what-it-is

Add support for cloud provider authentication methods to openFGA, so that operators can more easily deploy openFGA in a cloud environment.

# How it Works
[how-it-works]: #how-it-works

## New configuration options

The most straight forward option would be to introduce a new `datastore` string option `auth_method`. The user could then 
select from a list of supported authentication methods. If left blank, the default would be to use the existing username and password 
specified by the URI.

`-datastore.auth_method (string): "aws_rds_iam" | "gcp_cloud_iam" | etc`

Alternatively, a set of boolean flags could be used, but that seems cumbersome since only one would ever be used at a time.

`-datastore.aws_rds_iam_auth (bool)`

#### Inclusion of cloud provider SDKs configuration

Including the provider specific options, like `AWS_REGION`, seems unnecessary since the SDK's themselves are able to 
aqcuire these from the environment automatically. And adding them would make the configuration more complex and dependent on the SDKs. 

## Implementation

Authentication would need to be implemented for both databases supported by openFGA: postgres and mysql.

All the cloud provider connectors should return a `sql.DB`, so that adding new ones is simple as adding a new case:
```go
	var db *sql.DB
	switch cfg.AuthMethod {
	case "aws_rds_iam":
		db, err = openRDS(uri, cfg)
	case "":
		db, err = sql.Open("pgx", uri.String())
	default:
		return nil, fmt.Errorf("unsupported auth_method: %s", cfg.AuthMethod)
	}
```

### Platforms
#### AWS

AWS RDS IAM authentication is done by [acquiring an authentication token](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.html) 
from the AWS SDK and passing it as the password. This token is valid for 15 minutes, so it needs to be refreshed periodically.
This requires a background process to refresh the token within application.

#### GCP

GCP Cloud SQL Go Connector authentication is done via [a special dialer](https://cloud.google.com/sql/docs/postgres/samples/cloud-sql-postgres-databasesql-connect-connector).
This method is simpler than AWS RDS IAM, since it does not require a background process to refresh the token. I think [go-cloud](https://github.com/google/go-cloud)
provides this functionality out of the box, so that could be used. 

### Database connectors

Below are some examples of how the connectors could be implemented with AWS RDS IAM for both postgres and mysql.

#### Postgres

Since openFGA uses pgx, we can use the [BeforeConnect](https://github.com/jackc/pgx/blob/d626dfe94e250e911b77ac929e2be06f81042bd0/pgxpool/pool.go#L109)
hook to provide AWS RDS IAM tokens.

```go
func openRDS(uri *url.URL, cfg *sqlcommon.Config) (*sql.DB, error) {
	ctx := context.Background()
	iam, err := newIAM(ctx, cfg.Username, uri.Hostname(), uri.Port())
	if err != nil {
		return nil, err
	}
	conf, err := pgx.ParseConfig(uri.String())
	if err != nil {
		return nil, err
	}
	connector := pgxstd.GetConnector(*conf, pgxstd.OptionBeforeConnect(iam.BeforeConnect))
	db := sql.OpenDB(connector)
	return db, nil
}
```

#### Mysql

For mysql we need to wrap the sql connector to modify the connection string. This is done by implementing the [Driver interface](https://golang.org/pkg/database/sql/driver/#Driver).

```go
func (i *iam) Connect(ctx context.Context) (driver.Conn, error) {
	token, isNew, err := i.getRDSAuthToken(ctx)
	if err != nil {
		return nil, err
	}
	if !isNew {
		return i.baseConnector.Connect(ctx)
	}
	i.mysqlCfg.Passwd = token
	i.baseConnector, err = mysql.NewConnector(i.mysqlCfg)
	if err != nil {
		return nil, err
	}
	return i.baseConnector.Connect(ctx)
}

func (i *iam) Driver() driver.Driver {
	return i.baseConnector.Driver()
}

func open(...) (*sql.DB, error) {
	i := newIAM(...)
	return sql.OpenDB(i), nil
}
```
## Extensibility

This implementation would make it relatively simple to add new providers. You would just add a new case to `auth_method` and implement the connector.

# Migration
[migration]: #migration

There should not be any breaking changes just new configuration options.

# Drawbacks
[drawbacks]: #drawbacks
 - Adds dependencies to cloud provider SDKs.

# Alternatives
[alternatives]: #alternatives

- What other designs have been considered?
- Why is this proposal the best?
- What is the impact of not doing this?

# Prior Art
[prior-art]: #prior-art

### [go-cloud](https://github.com/google/go-cloud) 

Provides [IAM auth for GCP Cloud SQL](https://github.com/google/go-cloud/blob/master/postgres/gcppostgres/gcppostgres_test.go#L54), but not [AWS RDS](https://github.com/google/go-cloud/issues/3069). 
Does provide a unified API for connecting to various cloud providers. Could be used for inspiration on how to structure the code and GCP implementation.

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

