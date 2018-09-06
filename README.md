# Code
On my fork of Postgres, find the [code](https://github.com/melanieplageman/postgres/tree/experiment_null_in_subquery) for the hack discussed in the presentation case study

# Terminology
Terms I use in the presentation. Most definitions are gathered loosely from the
PostgreSQL documentation, unless otherwise specified.

### targetlist
A list of `TargetEntry` nodes which represent what the node projects. In the
presentation, I use this term largely to mean the target list of the top level
plan node, the `Result` node which represents the `SELECT` list of the query

### projection 
Relational algebra term meaning some subset of the attributes in a relation
optionally modified

###  `Result` node
Node used to project. If no outer plan, it evaluates a variable-free
targetlist. If there is an outer plan, it returns tuples from the outer plan
per its own targetlist. If `resconstantqual` is not NULL, the `Result` node is
a one-time qualification test that doesn't depend on any variables from the
outer plan, so it needs to be evaluated only once.

### tuple
A list of attributes each conforming to a data domain, or data type. For the purposes of this talk, it is basically a row in a table.

### relation
A set of tuples. I use the term loosely to refer to the heading and the body as well as the schema, basically as a table. [wikipedia](https://en.wikipedia.org/wiki/Relation_\(database\))

### predicate
An expression that evaluates to TRUE, FALSE, or UNKNOWN. Predicates are used in
the search condition of `WHERE` clauses and `HAVING` clauses, the join conditions
of FROM clauses, and other constructs where a Boolean value is required. They
are used in the `WHERE` clause as a filter specifying the criteria under which
each row will be affected 
[Microsoft docs](https://docs.microsoft.com/en-us/sql/t-sql/queries/predicates?view=sql-server-2017)

### qual
The qualification of a query. A boolean expression. The `WHERE` clause. Often used as a filter. 

### `Materialize` node
The output of the children of a material node is materialized into memory to make it available for rescan

### `jointree`
Table join tree representing the FROM and `WHERE` clauses

### `SubLink` node
Represents a subselect appearing in an expression, and in some cases also the
combining operator(s) just above it. It is non-executable and must be replaced
by a subplan during planning

# Useful links
Postgres' READMEs are all highly recommended. The planner README is especially good. [Optimizer README](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/README)

Read the [Postgres docs on SELECT](https://www.postgresql.org/docs/current/static/sql-select.html) for more details on the execution order of the different parts of a `SELECT` query.

# Planner functions
These are some of the important planning functions and what they do. These are good breakpoints to start with debugging.
```
PostgresMain
exec_simple_query()
	- execute a simple query
pg_analyze_and_rewrite()
	- takes raw parsetree and transforms into query tree
pg_plan_queries()
	- optimize normal plannable queries
planner()
	standard_planner()
		- query optimizer entry point for normal queries
		subquery_planner()
			Invokes the planner on a subquery. Is recursively called on each sub-SELECT found in the query tree

			does query tree pre-processing

			will attempt to turn WITH into initplans
			pull up subqueries into joins
			inline set returning functions
			flatten UNION ALLs
			expand inherited tables
			expression pre-processing on targetlist and quals and other expressions in the tree (for example within RTEs)
			remove redundant GROUP BY columns
			make outer joins into inner joins where possible


			calls grouping_planner()
				perform planning steps related to grouping and aggregation
				preprocess grouping sets
				preprocess group by
				preprocess targetlist

				calls query_planner()
					which generates path options which the grouping_planner can pick from
					constructs RelOptInfos for all base relations involved in the query
					build targetlists and make sure all the baserels have references to all needed vars
					add Restrict and Join clauses to appropriate lists belonging to	  these relations
					generate equivalence classes for provably equivalent expressions

					call generate_base_implied_equalities to generate RestrictInfos deduced from EquivalenceClasses

					once we have all equivalence sets merged, we can generate pathkeys which representing access methods and where each one can get you

					adds pages and some cost information
					
					calls make_one_rel() will join together all the paths into one rel	
						this does, amongst other things, join order
					converts all other parts of the query into pathlists
		calls set_cheapest to find the cheapest path
	choose final path
	create plan
```
