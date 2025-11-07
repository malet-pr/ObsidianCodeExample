
##### Full example

```java title:"Java 21 - Map" fold:true
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

```java title:"Java 21 - DTO" fold:true

public List<CustomerDTO> getCustomersByFilters(CustomerFilterDTO filter) {
    Map<String, String> nonNullParams = Arrays.stream(filter.getClass().getDeclaredFields())
        .peek(f -> f.setAccessible(true))
        .map(f -> {
            try {
                Object value = f.get(filter);
                return value != null && !(value instanceof String && ((String) value).isBlank())
                        ? Map.entry(f.getName(), value.toString())
                        : null;
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
        })
        .filter(Objects::nonNull)
        .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

    if (nonNullParams.isEmpty()) {
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
            case "startDate", "endDate" -> this.getByDynamicFilters(nonNullParams);
            default -> throw new WrongArgumentException("Invalid filter key: " + key);
        };
    } else {
        return this.getByDynamicFilters(nonNullParams);
    }
}
```

##### Extract non-null parameters from DTO to a Map

```java title:"Java 21 - DTO" fold:true
public static Map<String, String> nonNullFields(Object dto) {
    return Arrays.stream(dto.getClass().getDeclaredFields())
        .peek(f -> f.setAccessible(true))
        .map(f -> {
            try {
                Object value = f.get(dto);
                if (value == null) return null;
                if (value instanceof String && ((String) value).isBlank()) return null;
                return Map.entry(f.getName(), value.toString());
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
        })
        .filter(Objects::nonNull)
        .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
}
```

```java title:"Java 11 - DTO" fold:true
public static Map<String, String> nonNullFields(Object dto) {
    return Arrays.stream(dto.getClass().getDeclaredFields())
        .peek(f -> f.setAccessible(true))
        .map(f -> {
            try {
                Object value = f.get(dto);
                if (value == null) return null;
                if (value instanceof String && ((String) value).isBlank()) return null;
                // Use AbstractMap.SimpleEntry instead of Map.entry (available since Java 9, but safer)
                return new AbstractMap.SimpleEntry<>(f.getName(), value.toString());
            } catch (IllegalAccessException e) {
                throw new RuntimeException("Error accessing field: " + f.getName(), e);
            }
        })
        .filter(Objects::nonNull)
        .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
}
```


