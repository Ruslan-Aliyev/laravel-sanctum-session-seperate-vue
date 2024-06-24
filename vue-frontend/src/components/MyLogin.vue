<template>
    <div>
        <div class="row">
            <form action="#" @submit.prevent="handleLogin">
                <div class="form-row">
                    <input type="email" v-model="formData.email">
                </div>
                <div class="form-row">
                    <input type="password" v-model="formData.password">
                </div>
                <div class="form-row">
                    <button type="submit">Sign In</button>
                </div>
            </form>
        </div>
    </div>
</template>

<script>
import axios from 'axios'; 

axios.defaults.withCredentials = true;
axios.defaults.withXSRFToken = true;

export default {
    data() {
        return {
            formData: {
                email: '',
                password: ''
            }
        }
    },
    methods: {
        handleLogin() {
            axios.get('http://localhost:8000/sanctum/csrf-cookie').then(response => {
                console.log(response);
                axios.post('http://localhost:8000/login', this.formData).then(response => {
                    console.log('User signed in!');
                    axios.get('http://localhost:8000/api/protected-stuff').then(response => {
                        console.log(response.data);
                    });
                }).catch(error => console.log(error)); // credentials didn't match
            });
        }
    }
}
</script>