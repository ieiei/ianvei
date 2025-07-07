SparkSQL:
	Catalyst:
		Catalog:
			Table
				- name
				- schema
			StagingTableCatalog
				- stageCreate
				- stageReplace
				- stageCreateOrReplace
			TableCatalog
				- createTable
				- alterTable
				- dropTable
				- loadTable
				- listTables


Catalog extends  TableCatalog

Extension extends SparkSessionExtensions      


Source extends DataSourceRegister