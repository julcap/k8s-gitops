## Configuration

The secret for DB needs to be created manually. This is a temporary setup, it will eventually be moved to vault.

```
kubectl -n booklore create secret generic booklore-db --from-env-file=.env
```

.env 
```
root-password=super_secure_password
password=your_secure_password
```



