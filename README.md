# Interoperability API for Data Pump Files
For developers, database administrators and operations professionals who need interoperability and data exchange tools for Data Pump-type exports generated by Oracle®, the Interoperability API for Data Pump Files (`datapump`) is a Java® API for extracting information about compatible files, including metadata, table and row data without requiring external software. Unlike existing tools, this open-source project supplies a `javax.sql.DataSource` wrapper around an export file for running SQL queries against table data encoded within an export file using a low-footprint, in-memory [H2](http://www.h2database.com/html/main.html) database.

Under Java 8, the `datapump` project produces a ~70KB .jar file.

# Sample Test Cases
These annotated test methods use an export file named  `SCOTT.DMP` included in the project for testing purposes.

## Metadata
The following code parses the export file and validates the expected values for useful information about the export. This code completed in 0.005s.
```
	/**
	 * Test that we can extract the version and other details about the export.
	 * @throws Exception
	 */
	public void testScottMetadata() throws Exception {
		ClassLoader cl = Thread.currentThread().getContextClassLoader();
		try (TemporaryFile file = new TemporaryFile(cl.getResourceAsStream("scott.dmp"));) {
			DataPumpFile dumpFile = new DataPumpFile(file.toFile());
			assertEquals("Oracle 12c Release 1: 12.1.0", dumpFile.versionName());
			assertEquals("Wed May 23 14:34:07 EDT 2018", dumpFile.date().toString());
			assertEquals("AL32UTF8", dumpFile.characterSet());
			assertEquals(4096L, dumpFile.blockSize());
			assertEquals(true, dumpFile.master());
		}
	}
```

## Tables
The following code iterates through the file and collects the names of all exported tables. This code completed in 0.190s.
```
	/**
	 * Test the processing of tables using Java 8 Stream API methods.
	 * @throws Exception
	 */
	public void testScottTables() throws Exception {
		ClassLoader cl = Thread.currentThread().getContextClassLoader();
		try (TemporaryFile file = new TemporaryFile(cl.getResourceAsStream("scott.dmp"));) {

			DataPumpFile dumpFile = new DataPumpFile(file.toFile());
			String names = dumpFile.tables()
					.map(t -> t.get().name())
					.sorted()
					.collect(Collectors.joining(", "));
			assertEquals(names, "DEPT, EMP, SALGRADE");
		}
	}
```

## SQL Queries
The following code extracts the data from the `DEPT` and `EMP` tables and runs a query to extract the names of the six employees working in Chicago, Illinois (USA). This code completed in 0.498s.
```
	/**
	 * For an Oracle<sup>TM</sup>-type dump file (i.e., generated via <code>expdp</code>) can
	 * <pre>
	 * be parsed
	 * have selected tables (e.g., DEPT and EMP) loaded into an H2 in-memory database
	 * have SQL queries run against the exported table data
	 * that return correct results.
	 * </pre>
	 * @throws Exception
	 */
	public void testScott() throws Exception {
		ClassLoader cl = Thread.currentThread().getContextClassLoader();
		try (TemporaryFile file = new TemporaryFile(cl.getResourceAsStream("scott.dmp"));) {

			DataPumpFile dumpFile = new DataPumpFile(file.toFile());
			DataPumpDataSource dataSource = new DataPumpDataSource(dumpFile, "DEPT", "EMP");

			String sql = "SELECT ename"
				+ "  FROM emp e"
				+ "  JOIN dept d ON d.deptno = e.deptno"
				+ " WHERE d.loc = 'CHICAGO'"
				+ " ORDER BY 1";

			try (Connection connection = dataSource.getConnection();
				Statement statement = connection.createStatement();
				ResultSet rs = statement.executeQuery(sql);) {
				List<String> results = new ArrayList<>();
				while (rs.next()) {
					results.add(rs.getString(1));
				}
				assertEquals("[ALLEN, BLAKE, JAMES, MARTIN, TURNER, WARD]", results.toString());
			} catch (SQLException e) {
				e.printStackTrace();
			}
		}
	}
```

# Future Work
The existing Oracle tools are already very powerful, and this project should only supplement knowledge about integrating those features into safe, reliable and efficient deployment and archiving practices. The *Interoperability API for Data Pump Files* project sought to understand the code necessary for data exchange with uncompressed, unencrypted export files, especially to evaluate performance when extracting small amounts of information and where older or unpatched `impdp` versions prevented successful import (e.g., exported column data that was too wide for the column).

* Distribute through Maven Central
* Add reporting of test coverage
* Add Javadoc and other technical documentation about the API
* Add relevant examples
* Add sample files for multi-part (e.g., parallel) exports
* Add sample files for testing character set conversion in data and metadata

# Related Tools

* [InfiniTest](https://infinitest.github.io/)

# License
This software is available under the terms of the MIT License (MIT).

# Credits
All trademarks, registered trademarks and products mentioned remain the property of their respective owners.

Java and Oracle are registered trademarks of Oracle Corporation and/or its affiliates.
Apache Maven™ is a trademark of The Apache Software Foundation.
