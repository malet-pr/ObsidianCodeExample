```java title:"Java 21" fold:true
    public List<CustomerDTO> getCustomersByFilters(Map<String, String> queryParams)  {
        Map<String, String> nonNullParams = queryParams.entrySet().stream()
                .filter(e -> e.getValue() != null && !e.getValue().isBlank())
                .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
        if (nonNullParams.isEmpty()){
            throw new WrongArgumentException("At least one filter parameter must be provided");
        } else if (nonNullParams.size() == 1) {
            Map.Entry<String, String> param = nonNullParams.entrySet().iterator().next();
            String key = param.getKey();
            String value = param.getValue();
            return switch (key) {
                case "code" -> List.of(this.getByCode(value));
                case "regionCode" -> this.getByRegionCode(value);
                case "status" -> this.getByStatus(value);
                case "verified" -> this.getByVerified(value);
                case "active" -> this.getAllByActive(value);
                case "startDate","endDate" -> this.getByDynamicFilters(nonNullParams);
                default -> throw new WrongArgumentException("Invalid filter key: " + key);
            };
        } else {
            return this.getByDynamicFilters(nonNullParams);
        }
    }
```