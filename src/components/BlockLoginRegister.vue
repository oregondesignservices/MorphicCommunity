<template>
  <b-jumbotron class="mb-0" bg-variant="light" id="login-register" role="main">
    <b-row>
      <b-col md="1"></b-col>
      <b-col md="4" class="ml-md-2 mr-lg-5">
        <h2 class="mb-3" id="community-heading">Register Group</h2>
        <UserRegistration />
      </b-col>
      <b-col md="4" class="pl-md-5 pr-lg-5">
        <UserLogin />
      </b-col>
      <b-col md="2"></b-col>
    </b-row>
  </b-jumbotron>
</template>

<style>
  #login-register {
    padding: 2rem 2rem;
    background: url(/img/login.jpg) no-repeat left center;
  }
</style>

<script>
import UserLogin from "@/components/UserLogin";
import UserRegistration from "@/components/UserRegistration";

export default {
    name: "BlockLoginRegister",
    components: {
        UserLogin,
        UserRegistration
    }
};
</script>
