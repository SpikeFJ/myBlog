# 一. 问题
1.DaoImpl有如下代码,查询结果显示`ClientMonitorRecordEntity`
中`b2cUserCode`和`createDateTime`值为null
```java
	List<ClientMonitorRecordEntity> list = dalClient.queryForList("logClientMonitor.query_client_monitor_record_by_conditions",
param, ClientMonitorRecordEntity.class);
```
`ClientMonitorRecordEntity`类结构如下
```java
	/**
	 * 主键
	 */
	private Integer id;

	/**
	 * 易购号
	 */
	private String b2cUserCode;

    	/**
	 * 创建时间。格式：YYYY-MM-DD HH:mm:ss
	 */
	private String createDateTime;

    @Column(name = "b2c_user_code")
	public String getB2cUserCode() {
		return b2cUserCode;
	}

	@Column(name = "create_datetime")
	public String getCreateDateTime() {
		return createDateTime;
	}
    .....
```

二. 定位

1. `org.springframework.jdbc.core.JdbcTemplate`
```java
public <T> T query(
			PreparedStatementCreator psc, final PreparedStatementSetter pss, final ResultSetExtractor<T> rse)
			throws DataAccessException {

		Assert.notNull(rse, "ResultSetExtractor must not be null");
		logger.debug("Executing prepared SQL query");

		return execute(psc, new PreparedStatementCallback<T>() {
			public T doInPreparedStatement(PreparedStatement ps) throws SQLException {
				ResultSet rs = null;
				try {
					if (pss != null) {
						pss.setValues(ps);
					}
					rs = ps.executeQuery();
					ResultSet rsToUse = rs;
					if (nativeJdbcExtractor != null) {
						rsToUse = nativeJdbcExtractor.getNativeResultSet(rs);
					}
					//主要关注这里结果集处理
					return rse.extractData(rsToUse);
				}
				finally {
					JdbcUtils.closeResultSet(rs);
					if (pss instanceof ParameterDisposer) {
						((ParameterDisposer) pss).cleanupParameters();
					}
				}
			}
		});
	}
```

2. `org.springframework.jdbc.core.RowMapperResultSetExtractor`
```java
public List<T> extractData(ResultSet rs) throws SQLException {
		List<T> results = (this.rowsExpected > 0 ? new ArrayList<T>(this.rowsExpected) : new ArrayList<T>());
		int rowNum = 0;
		while (rs.next()) {
			//子类BeanPropertyRowMapper处理映射
			results.add(this.rowMapper.mapRow(rs, rowNum++));
		}
		return results;
	}
```

3. `org.springframework.jdbc.core.BeanPropertyRowMapper`

```java
public T mapRow(ResultSet rs, int rowNumber) throws SQLException {
		Assert.state(this.mappedClass != null, "Mapped class was not specified");
		T mappedObject = BeanUtils.instantiate(this.mappedClass);
		BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(mappedObject);
		initBeanWrapper(bw);

		ResultSetMetaData rsmd = rs.getMetaData();
		int columnCount = rsmd.getColumnCount();
		Set<String> populatedProperties = (isCheckFullyPopulated() ? new HashSet<String>() : null);

		for (int index = 1; index <= columnCount; index++) {
			//读取数据库字段
			String column = JdbcUtils.lookupColumnName(rsmd, index);
			//从mappedFields中根据数据库字段获取对应的类字段信息，可以发现循环到b2cUserCode和createDateTime时pd都是null
			//跟踪mappedFields发现是初始化时赋值的，参见步骤4
			PropertyDescriptor pd = this.mappedFields.get(column.replaceAll(" ", "").toLowerCase());
			if (pd != null) {
				try {
					Object value = getColumnValue(rs, index, pd);
					if (logger.isDebugEnabled() && rowNumber == 0) {
						logger.debug("Mapping column '" + column + "' to property '" +
								pd.getName() + "' of type " + pd.getPropertyType());
					}
					try {
						bw.setPropertyValue(pd.getName(), value);
					}
					catch (TypeMismatchException e) {
						if (value == null && primitivesDefaultedForNullValue) {
							logger.debug("Intercepted TypeMismatchException for row " + rowNumber +
									" and column '" + column + "' with value " + value +
									" when setting property '" + pd.getName() + "' of type " + pd.getPropertyType() +
									" on object: " + mappedObject);
						}
						else {
							throw e;
						}
					}
					if (populatedProperties != null) {
						populatedProperties.add(pd.getName());
					}
				}
				catch (NotWritablePropertyException ex) {
					throw new DataRetrievalFailureException(
							"Unable to map column " + column + " to property " + pd.getName(), ex);
				}
			}
		}

		if (populatedProperties != null && !populatedProperties.equals(this.mappedProperties)) {
			throw new InvalidDataAccessApiUsageException("Given ResultSet does not contain all fields " +
					"necessary to populate object of class [" + this.mappedClass + "]: " + this.mappedProperties);
		}

		return mappedObject;
	}
```
4. mappedFields初始化
```java
	protected void initialize(Class<T> mappedClass) {
		this.mappedClass = mappedClass;
		this.mappedFields = new HashMap<String, PropertyDescriptor>();
		this.mappedProperties = new HashSet<String>();
		//反射类中所有字段
		PropertyDescriptor[] pds = BeanUtils.getPropertyDescriptors(mappedClass);
		for (PropertyDescriptor pd : pds) {
			if (pd.getWriteMethod() != null) {
				//mappedFields中添加 小写原始字段字符串,注意这一步，涉及到后续解决方法
				this.mappedFields.put(pd.getName().toLowerCase(), pd);
				//根据字段名称处理，如果处理后的字段值不等于字段名称，则添加处理后的值
				String underscoredName = underscoreName(pd.getName());
				if (!pd.getName().toLowerCase().equals(underscoredName)) {
					this.mappedFields.put(underscoredName, pd);
				}
				this.mappedProperties.add(pd.getName());
			}
		}
	}


	private String underscoreName(String name) {
		StringBuilder result = new StringBuilder();
		if (name != null && name.length() > 0) {
			//1.将第一个字符转成小写
			result.append(name.substring(0, 1).toLowerCase());
			//2.从第二个字符开始遍历
			for (int i = 1; i < name.length(); i++) {
				String s = name.substring(i, i + 1);
				//3.如果字符 = 该字符的大写，注意此处如果是数字则会判断为true
				if (s.equals(s.toUpperCase())) {
					//4.添加“_”,并将大写转换为小写
					result.append("_");
					result.append(s.toLowerCase());
				}
				else {
					result.append(s);
				}
			}
		}
		return result.toString();
	}
```

## 总结
`underscoreName`解释了问题的原因：**因为数据库字段默认都是“hello_world”这种命名，underscoreName本意是将所有大写字符转换成"_"+小写字符。**

但是形如`b2cUserCode`这种带数字的字符，在判断是否大写的时候，会导致误判，导致转换成`b_2c_user_code`,


而`createDateTime`则转换成`create_date_time`和数据库字段`create_datetime`不一致。
我认为datetime是一个单词，所以数据库命名时没有采用`create_date_time`


## 解决办法
1. 通过重新命名符合spring规则的字段
2. 修改sql，添加别名 如select b_2c_user_code as b2cUserCode
之所以这样可以生效，是因为上述代码中有这样一段
```java
this.mappedFields.put(pd.getName().toLowerCase(), pd);
```
所以b2cUserCode在mappedFields中可以找到。

## 思考
`underscoreName`这种转换方式太固定，肯定会遇到不一致的情况。原以为spring会解析字段上的注解采用注解上的name来映射，但实际却不是这样。具体原因是什么呢？


